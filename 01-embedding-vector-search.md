# 1. 임베딩(벡터) 유사도 검색

## 가능 여부: ✅ 가능 (v3.4+)

StarRocks는 v3.4부터 **근사 최근접 이웃 검색(ANNS, Approximate Nearest Neighbor Search)** 을 위한 벡터 인덱스를 지원합니다. 임베딩 벡터를 `ARRAY<FLOAT>` 컬럼에 저장하고, 인덱스를 걸어 top-k 유사 벡터를 빠르게 검색합니다.

- 지원 클러스터: shared-nothing 클러스터 (v3.4 이상)
- 상태: **베타/실험 기능** → FE 설정 `enable_experimental_vector = true` 필요
- 인덱싱 단위: Segment 파일 레벨 (각 검색 아이템 → row ID 매핑)

## 인덱스 종류

| 인덱스 | 방식 | 압축비 | 특징 |
|--------|------|:-----:|------|
| **HNSW** | 계층적 그래프 (Hierarchical Navigable Small World) | ~1:0.8 | 정밀·낮은 지연시간. 지연 민감 워크로드 적합 |
| **IVFPQ** | 클러스터링 + Product Quantization | ~1:0.15 | 고압축·대규모. fine-ranking 필요, 연산비 높음 |

## 거리(유사도) 메트릭

- `l2_distance` — 유클리드 거리 (작을수록 유사)
- `cosine_similarity` — 코사인 유사도 (클수록 유사)

## 인덱스 생성 예시

```sql
CREATE TABLE example (
    id BIGINT NOT NULL,
    vector ARRAY<FLOAT> NOT NULL,
    INDEX idx_vec (vector) USING VECTOR (
        "index_type"      = "hnsw",           -- 또는 "ivfpq"
        "dim"             = "5",
        "metric_type"     = "l2_distance",    -- 또는 "cosine_similarity"
        "is_vector_normed"= "false",
        "M"               = "16",              -- HNSW 전용
        "efconstruction"  = "40"               -- HNSW 전용
    )
) ENGINE=OLAP
DUPLICATE KEY(id)
DISTRIBUTED BY HASH(id);
```

- IVFPQ 전용 파라미터: `nlist`, `nbits`, `M_IVFPQ`

## 검색 쿼리 예시

```sql
-- 근사 검색 (인덱스 사용) — approx_* 함수 사용
SELECT id,
       approx_l2_distance([1,2,3,4,5], vector) AS dist
FROM   example
ORDER  BY approx_l2_distance([1,2,3,4,5], vector) ASC
LIMIT  10;

-- 코사인 메트릭이면 approx_cosine_similarity 사용
-- 정밀 계산(인덱스 미사용): l2_distance / cosine_similarity

-- 검색 정확도 튜닝 힌트
SELECT /*+ SET_VAR (ann_params='{efsearch=256}') */ id
FROM   example
ORDER  BY approx_l2_distance([1,2,3,4,5], vector) ASC
LIMIT  10;
```

## 제약사항

- **테이블당 벡터 인덱스는 1개만** 지원
- FE 설정 `enable_experimental_vector = true` 필요
- 베타 단계 → 프로덕션 도입 시 버전·안정성 검증 권장
- GPU 가속은 (조사 시점 기준) 예정 기능
