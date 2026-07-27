# 3. 하이브리드 검색 (벡터 + 전문 + 스칼라)

## 가능 여부: ✅ 가능 — 하나의 SQL로 결합

StarRocks의 가장 큰 강점은 **벡터 유사도 검색 + 키워드(전문) 매칭 + 정형 필터(WHERE)** 를 별도 시스템 없이 **단일 SQL**로 처리할 수 있다는 점입니다. 즉, "의미 유사 + 키워드 일치 + 조건 제약"을 한 번에 조인/필터/랭킹할 수 있습니다.

> Alibaba Cloud EMR Serverless StarRocks 문서에서는 이를 full-text(BM25 + 역색인) 채널과 vector(HNSW/IVFPQ/DiskANN) 채널이 **병렬 recall** 후 엔진 내부에서 **RRF / 가중 융합 / learning-to-rank**로 통합·스코어링하는 "삼중 융합(three-way fused)" 검색으로 소개합니다.

## 패턴 A — 벡터 유사도 + 스칼라/메타데이터 필터 (지금 바로 가능)

```sql
SELECT d.id, d.chunk_text,
       cosine_similarity([0.12, 0.45, 0.67, ...], d.embedding) AS sim
FROM   document_embeddings d
WHERE  d.tenant_id = 12345
  AND  json_extract(d.metadata, '$.category') = 'technical'
  AND  d.created_at >= '2024-01-01'
ORDER  BY sim DESC
LIMIT  20;
```

- 파티션 프루닝(`tenant_id`), JSON 필터(category), 날짜 범위가 벡터 검색 **이전에** 후보를 줄여줌
- StarRocks의 "In-Filter 딥 최적화"는 스칼라 필터와 벡터 검색을 동기 실행해 recall을 30%+ 개선한다고 소개됨

## 패턴 B — 벡터 + 전문 검색 결합 (키워드 + 의미)

```sql
-- 벡터 인덱스(01번 문서) + GIN 역색인(02번 문서)을 같은 테이블/조인에 구성
SELECT id, chunk_text,
       approx_cosine_similarity([...], embedding) AS sim
FROM   documents
WHERE  content MATCH_ANY 'starrocks, vector'   -- 전문(키워드) 채널
ORDER  BY sim DESC                              -- 벡터(의미) 채널
LIMIT  10;
```

## 패턴 C — 가중치 융합 (weighted fusion)

여러 신호를 SQL 표현식으로 직접 가중 결합:

```sql
-- 예: 인기도 × 벡터 유사도
SELECT i.id,
       i.popularity_score * (1 - cosine_distance(u.embedding, i.embedding))
         AS recommendation_score
FROM   items i, user_vec u
ORDER  BY recommendation_score DESC
LIMIT  20;
```

## 패턴 D — RAG 파이프라인 (조인 + 권한 + 벡터)

```sql
WITH relevant_chunks AS (
    SELECT d.id, d.chunk_text,
           cosine_distance(d.embedding, ${query_embedding}) AS dist
    FROM   document_embeddings d
    JOIN   document_permissions p ON d.document_id = p.document_id
    WHERE  p.user_id = ${user_id}
      AND  p.permission_type = 'read'
    ORDER  BY dist ASC
    LIMIT  50
)
SELECT chunk_text, dist FROM relevant_chunks ORDER BY dist ASC LIMIT 10;
```

## 융합(fusion) 방식 정리

| 방식 | 현재 가능성 | 방법 |
|------|:----------:|------|
| **가중치 융합** | ✅ 지금 가능 | SQL 표현식으로 `벡터점수 × 가중치` / 여러 점수 선형결합 |
| **스칼라 프리필터 + ANN** | ✅ 지금 가능 | WHERE 절 필터 후 ANN (In-Filter 최적화) |
| **RRF (Reciprocal Rank Fusion)** | ⚠️ 최신 버전 | 엔진 내장 RRF는 EMR Serverless / v4.x 개선판에서 소개. 순수 OSS 버전은 확인 필요 |
| **learning-to-rank** | ⚠️ 최신 버전 | 최신 배포판 기능으로 소개됨 |

## StarRocks vs 전용 벡터 DB

**StarRocks가 유리한 경우**
- 벡터 + 분석(SQL) 워크로드 혼합
- 임베딩과 메타데이터 간 복잡한 조인 필요
- 실시간 수집 후 즉시 조회
- 벡터/검색/분석 시스템을 하나로 통합해 운영 복잡도 절감

**전용 벡터 DB(Milvus 등)를 고려할 경우**
- 순수 벡터 유사도만 필요한 경우
- 특수 벡터 알고리즘이 필요한 경우
- 관리형 클라우드 서비스 선호

## 종합 결론

- **벡터 + 스칼라 + 전문 검색을 단일 SQL로 결합하는 하이브리드 검색은 가능**합니다.
- 지금 바로 확실하게 쓸 수 있는 방식: **스칼라 프리필터 + ANN**, **가중치 융합(SQL 표현식)**.
- 엔진 내장 **BM25 스코어링 / RRF / learning-to-rank** 는 최신 버전(v4.1+ 및 특정 클라우드 배포판)에서 강화되고 있으므로, **도입 전 사용 버전과 기능 지원 여부를 반드시 확인**하는 것이 좋습니다.

## ⚠️ 오픈소스 4.1.1 기준 명확화

- 위 "엔진 내장 RRF / 가중융합 / learning-to-rank / BM25"는 **Alibaba Cloud EMR Serverless StarRocks 등 상용/클라우드 배포판** 소개 내용이며, **오픈소스 4.1.1에는 없습니다.**
- OSS 4.1.1에서 하이브리드는 **직접 조립**해야 합니다:
  - 패턴 1: `WHERE MATCH(...) ORDER BY approx_cosine_similarity(...)` — 전문으로 후보 좁히고 벡터로 정렬 (엄밀히는 필터+랭킹).
  - 패턴 2: 벡터 top-N 쿼리 + 전문 top-N 쿼리를 각각 실행 → **앱 코드에서 RRF 융합**.
- 융합 동작 원리와 OSS 구현 구조는 → [`04-fusion-and-oss-implementation.md`](./04-fusion-and-oss-implementation.md)
- 세 방식의 성능 비교는 → [`05-performance-comparison.md`](./05-performance-comparison.md)
