# 참고 출처

> 조사일: 2026-07-24

## StarRocks 공식 문서
- [Vector Index | StarRocks](https://docs.starrocks.io/docs/table_design/indexes/vector_index/) — 벡터 인덱스(HNSW/IVFPQ), ANN, 거리 메트릭, SQL 문법. **"shared-nothing only" 명시**
- [Full-text inverted index | StarRocks](https://docs.starrocks.io/docs/table_design/indexes/inverted_index/) — GIN 역색인, 토크나이저(parser), MATCH 계열 함수
- [Indexes | StarRocks](https://docs.starrocks.io/docs/category/indexes/)
- [Architecture | StarRocks](https://docs.starrocks.io/docs/introduction/Architecture/) — FE/BE/CN 구조, FE 역할(Leader/Follower/Observer) (2026-07-27 확인)
- [Feature Support: Shared-data Clusters | StarRocks](https://docs.starrocks.io/docs/deployment/shared_data/feature-support-shared-data/) — shared-data 지원 기능 목록, 벡터 인덱스 없음 (2026-07-27 확인)
- [Data distribution | StarRocks](https://docs.starrocks.io/docs/table_design/data_distribution/) — 파티션 + 해시 버킷(Tablet) 행 단위 분산 (2026-07-27 확인)
- [Resource group | StarRocks](https://docs.starrocks.io/docs/administration/management/resource_management/resource_group/) — BE 논리적 워크로드 격리

## GitHub 이슈 (기능/로드맵)
- [[Feature] Support vector index and ANNS · #46678](https://github.com/StarRocks/starrocks/issues/46678)
- [[Feature] Full-Text Search Capability Development & Optimization Plan · #61827](https://github.com/StarRocks/starrocks/issues/61827) — BM25 등 전문 검색 개선 플랜 (2026-07-27 재확인: BM25 항목 없음, tokenize/match_any/match_all 완료)
- [StarRocks Roadmap 2026 · #67632](https://github.com/StarRocks/starrocks/issues/67632) — shared-data용 벡터 인덱스 계획 항목만 존재 (2026-07-27 확인)
- [Releases · StarRocks/starrocks](https://github.com/StarRocks/starrocks/releases) — 4.1.1(2026-06-18)이 최신, 이후는 하위 브랜치 버그픽스 (2026-07-27 확인)
- [Support tokenize function · #45145](https://github.com/StarRocks/starrocks/issues/45145)
- [Support Inverted Indexes for Primary Key Tables · #56876](https://github.com/StarRocks/starrocks/issues/56876)

## 하이브리드 검색 / 활용 사례
- [Alibaba Cloud EMR Serverless StarRocks Lakehouse Multimodal Retrieval: One SQL for Full-Text, Scalar, and Vector Hybrid Search](https://www.alibabacloud.com/blog/alibaba-cloud-emr-serverless-starrocks-lakehouse-multimodal-retrieval-one-sql-for-full-text-scalar-and-vector-hybrid-search_603345)
- [The Complete Guide to Hybrid Search (Alibaba Cloud)](https://www.alibabacloud.com/blog/the-complete-guide-to-hybrid-search-the-perfect-blend-of-full-text-and-vector-search_602921)
- [StarRocks as a Vector Database: HNSW, Hybrid Queries & When to Use It (JusDB)](https://www.jusdb.com/blog/starrocks-as-a-vector-database-architecture-scaling-and-best-practices)
- [StarRocks Vector Search: Production-Ready Setup in 6 Steps (Necati Demir)](https://n.demir.io/articles/starrocks-vector-search-production-ready-setup-in-6-steps/)
- [Full Text Research in StarRocks (Fresha, Medium)](https://medium.com/fresha-data-engineering/full-text-research-in-starrocks-213fb3e3a2e7)

## 참고 개념
- [Reciprocal Rank Fusion Explained (Serghei's Blog)](https://blog.serghei.pl/posts/reciprocal-rank-fusion-explained/)
