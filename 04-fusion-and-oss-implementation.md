# 4. 하이브리드 융합 원리 & OSS 4.1.1 구현 구조

## 핵심 질문: 전체를 합산하나, 아니면 각 채널 top-N을 뽑아 합치나?

**답: 각 채널이 먼저 top-N을 뽑고(recall), 그 후보 합집합에만 융합 점수를 매겨 재정렬한다.**
전체 데이터(corpus)에 하이브리드 점수를 전부 계산하지 않습니다.

## 표준 하이브리드 동작 흐름

```
                       쿼리
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
      [토크나이저 채널]        [임베딩 채널]
       BM25로 top-N           벡터 유사도로 top-N
       (예: top-100)          (예: top-100)
             │                     │
             └──────────┬──────────┘
                        ▼
             ② 두 후보군 합집합 (중복 제거)
                        ▼
             ③ 후보에만 융합 점수 계산 (RRF or 가중합)
                        ▼
             ④ 재정렬 후 최종 top-K 반환 (예: top-10)
```

## 왜 "전체 합산"이 아닌가

1. **비용** — 문서 수억 건에 매 쿼리마다 BM25 + 벡터 유사도를 전부 계산하는 것은 비현실적. 각 채널이 인덱스(역색인 / HNSW)로 상위 후보만 빠르게 recall.
2. **점수 스케일 불일치(score-incompatibility)** — BM25는 상한 없음(0~수십), 코사인 유사도는 0~1. 그냥 더하면 스케일 큰 쪽이 지배. 그래서 단순 합산은 위험.

## 융합 방식 2가지

### A. RRF (Reciprocal Rank Fusion) — 점수 대신 "순위"로 합침
```
score(doc) = Σ  1 / (k + rank_channel)     (보통 k = 60)
```
- 예: 어떤 문서가 토크나이저 3등 + 벡터 5등 → `1/(60+3) + 1/(60+5)`
- **장점**: 점수 스케일 안 맞아도 안전, 튜닝 거의 불필요.
- Elasticsearch / OpenSearch / Weaviate / Qdrant / Azure AI Search 등 대부분이 기본 채택.

### B. 가중 점수 합산 (weighted fusion) — 점수를 정규화 후 합침
```
score = α × norm(BM25) + (1 - α) × norm(cosine)
```
- 점수를 0~1로 정규화 후 가중치 α로 결합.
- **장점**: 세밀 조정 가능. **단점**: 정규화·α 튜닝 필요, 데이터 바뀌면 재조정.

## ⚠️ 오픈소스 StarRocks 4.1.1에서의 현실

**OSS 4.1.1에는 BM25 스코어링·RRF·가중융합 엔진이 없습니다.** 위 그림의 ②~④(융합·재정렬)를 **직접 구현**해야 합니다. 두 가지 구현 패턴:

### 패턴 1 — SQL 한 방 (필터 + 랭킹, 엄밀히는 "융합" 아님)
전문으로 후보를 좁히고 벡터로 정렬. 가장 단순하고 확실.
```sql
SELECT id, content,
       approx_cosine_similarity([...], embedding) AS sim
FROM   documents
WHERE  content_tokenized MATCH_ANY '검색 키워드'   -- ① 전문 채널로 프리필터
ORDER  BY sim DESC                                 -- 벡터 유사도로 랭킹
LIMIT  10;
```
- 장점: 쿼리 1번, 빠름. 단점: 진짜 RRF가 아니라 "전문 필터 통과분 중 벡터순 정렬"이라 두 신호의 균형 조절은 안 됨.

### 패턴 2 — 진짜 RRF (recall만 StarRocks, 융합은 앱에서)
```
1) 벡터 채널 쿼리:   ORDER BY approx_cosine_similarity(...) LIMIT 100   → 벡터 top-100
2) 전문 채널 쿼리:   WHERE ... MATCH_ANY '...' LIMIT 100                → 전문 top-100
3) 애플리케이션 코드에서 두 결과의 rank로 RRF 합산 → 최종 top-10
```
- StarRocks는 ①(recall)만 담당, ②~④(융합·재정렬)는 **앱 코드**가 담당.
- RRF/가중치 로직을 직접 짜야 하지만, 두 채널 균형을 제대로 조절 가능.

### 의사코드 (앱단 RRF)
```python
def rrf(vec_hits, text_hits, k=60, top=10):
    scores = {}
    for rank, doc in enumerate(vec_hits, 1):
        scores[doc.id] = scores.get(doc.id, 0) + 1/(k+rank)
    for rank, doc in enumerate(text_hits, 1):
        scores[doc.id] = scores.get(doc.id, 0) + 1/(k+rank)
    return sorted(scores, key=scores.get, reverse=True)[:top]
```

## 정리
- 하이브리드 = **각 채널 top-N recall → 후보 합집합 → RRF/가중합으로 재정렬**. 전체 합산 아님.
- **OSS 4.1.1**: recall은 StarRocks가, **융합·재정렬은 직접 구현**(SQL 필터+정렬 또는 앱단 RRF).
- 엔진 내장 자동 융합(RRF/LTR)이 필요하면 상용/클라우드 배포판(EMR Serverless 등) 검토.
