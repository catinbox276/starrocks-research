# StarRocks: 임베딩 유사도 검색 · 토크나이저 검색 · 하이브리드 검색 리서치

> 조사일: 2026-07-24
> 기준 버전: **오픈소스(OSS) StarRocks 4.1.1**
> 주제: StarRocks DB에서 (1) 임베딩(벡터) 유사도 검색, (2) 토크나이저 기반 전문(full-text) 검색, (3) 두 가지를 결합한 하이브리드 검색이 가능한지

## 결론 요약 (TL;DR) — 오픈소스 4.1.1 기준

| 항목 | 가능 여부 | 비고 |
|------|:--------:|------|
| **임베딩(벡터) 유사도 검색** | ✅ 가능 | HNSW / IVFPQ, `approx_*` 함수. v3.4+. **여전히 실험(experimental) 기능** |
| **토크나이저 전문 검색** | ✅ 가능 | GIN 역색인, `MATCH`/`MATCH_ANY`/`MATCH_ALL`. v3.3+. 4.1부터 shared-data도 지원(Beta) |
| **하이브리드 검색** | ⚠️ **직접 조립만 가능** | "스칼라/전문 필터 + 벡터 정렬"은 SQL로 가능. **BM25 랭킹·RRF 자동 융합은 OSS에 없음** |
| **한국어 형태소 토크나이저** | ❌ 없음 | `standard` 파서로 색인은 되나 조사/어미 처리 안 됨 → 외부 형태소 전처리 필요 |

### 오픈소스에서 꼭 알아야 할 3가지
1. **BM25 스코어링 / RRF / 가중융합 / learning-to-rank 엔진은 오픈소스 4.1.1에 없습니다.** 이 기능들은 Alibaba Cloud EMR Serverless StarRocks 등 **상용/클라우드 배포판** 소개 내용입니다.
2. **벡터 인덱스는 실험 기능** — FE 설정 `enable_experimental_vector = true` 필요, 테이블당 벡터 인덱스 1개 제한.
3. **한국어 전용 토크나이저 없음** — 제대로 된 한글 전문 검색은 외부 형태소 분석(mecab-ko/Nori 등) 전처리가 사실상 필수.

## 문서 구성

- [`01-embedding-vector-search.md`](./01-embedding-vector-search.md) — 벡터 인덱스 & 임베딩 유사도 검색
- [`02-tokenizer-fulltext-search.md`](./02-tokenizer-fulltext-search.md) — 토크나이저 & 전문 역색인 검색 (+ 한글 지원 한계 & 외부 전처리)
- [`03-hybrid-search.md`](./03-hybrid-search.md) — 하이브리드 검색 결합 방법
- [`04-fusion-and-oss-implementation.md`](./04-fusion-and-oss-implementation.md) — **하이브리드 융합 원리(RRF/가중합) & OSS 4.1.1 구현 구조**
- [`05-performance-comparison.md`](./05-performance-comparison.md) — 토크나이저 vs 임베딩 vs 하이브리드 성능 비교
- [`06-current-environment-feasibility.md`](./06-current-environment-feasibility.md) — **현재 운영 환경(FE+CN) 임베딩 검색 가능 여부 조사 (조사일 2026-07-27): 불가능, FE+BE 필요**
- [`starrocks-architecture.drawio`](./starrocks-architecture.drawio) — 아키텍처 다이어그램 (2계층 구조 / 확장 비교 / 데이터 분배)
- [`sources.md`](./sources.md) — 참고 출처

## 버전 메모 (4.1 / 4.1.1)
- **4.1** (2026-05-29): 검색 관련 신규는 `[Beta] Inverted Index on shared-data`(#66541) — 역색인이 shared-data 클러스터로 확장.
- **4.1.1**: 버그 픽스 패치 (컨테이너 환경에서 BE 프로세스 기동 안정성 수정). **검색 기능은 4.1.0과 동일.**
