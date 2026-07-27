# 2. 토크나이저 기반 전문(Full-Text) 검색

## 가능 여부: ✅ 가능 (v3.3+)

StarRocks는 v3.3.0부터 **GIN(Generalized Inverted Index, 역색인)** 기반의 전문 검색을 지원합니다. 텍스트를 토크나이저로 단어 단위로 쪼갠 뒤, `단어 → row number` 매핑을 색인해 키워드로 해당 행을 빠르게 찾습니다.

예) `"hello world"`(row 123) → `hello -> 123`, `world -> 123`

## 지원 버전 & 구현체

| 구현체 | 버전 | 비고 |
|--------|:----:|------|
| CLucene 기반 (기본) | v3.3.0+ | 기본 구현 |
| Built-in 구현 | v4.1.0+ | shared-nothing & shared-data 모두 지원 (`imp_lib=builtin`) |
| Primary Key 테이블 지원 | v4.0+ | |

## 지원 토크나이저 (parser)

| parser | 설명 |
|--------|------|
| `none` (기본) | 토큰화 안 함. 행 전체를 하나의 색인 아이템으로. `LIKE`, 비교 연산자(`=`,`IN` 등) 사용 가능 |
| `english` | 비알파벳 문자 기준 분리, 대문자 → 소문자 변환 |
| `chinese` | CJK Analyzer 사용 (중국어 토큰화) |
| `standard` | Unicode Text Segmentation 기반 다국어 토큰화, 혼용 언어 처리 |

## 인덱스 생성 예시

```sql
-- 테이블 생성 시
CREATE TABLE t (
    k BIGINT NOT NULL,
    v STRING,
    INDEX idx (v) USING GIN("parser" = "english")
) ENGINE=OLAP
DUPLICATE KEY(k)
DISTRIBUTED BY HASH(k) BUCKETS 1
PROPERTIES ("replicated_storage" = "false");

-- 테이블 생성 후 추가
ALTER TABLE t ADD INDEX idx (v) USING GIN('parser' = 'english');
CREATE INDEX idx ON t (v) USING GIN('parser' = 'english');

-- Built-in 구현 (v4.1+)
INDEX gin_idx (col) USING GIN("parser" = "english", "imp_lib" = "builtin")
```

## 검색 함수

**토큰화 사용 시 (`parser` != `none`):**

```sql
SELECT * FROM t WHERE v MATCH '%keyword%';          -- 와일드카드 매칭
SELECT * FROM t WHERE v MATCH_ANY 'word1, word2';   -- OR (하나라도 포함)
SELECT * FROM t WHERE v MATCH_ALL 'word1, word2';   -- AND (모두 포함)
```

**토큰화 미사용 (`parser = 'none'`):** `LIKE`, `MATCH`(=LIKE), `MATCH_ANY`, `MATCH_ALL`, `=`, `!=`, `<=`, `>=`, `IN`, `NOT IN`, `IS NOT NULL`

## 제약사항 / 유의점

- English/standard 토크나이저에서 **검색 키워드는 소문자**여야 함
- `%` 퍼지 매칭은 전체 단어가 아닌 **단어 일부**와 매칭됨
- 검색 술어(predicate)는 **WHERE 절에 위치**해야 함 (pushdown 조건)
- **BM25 관련성 랭킹**은 v3.3 역색인 공식 문서에 명시되어 있지 않음 → `MATCH` 계열은 기본적으로 불리언 필터 성격. 본격 BM25 스코어링은 v4.1+ Full-Text Search 개선 플랜(GitHub #61827)에서 진행 중이므로 사용 버전 확인 필요 (**오픈소스 4.1.1에는 네이티브 BM25 없음**)

## ⭐ 한국어(한글) 지원

**StarRocks에는 한국어 전용 토크나이저(형태소 분석기)가 없습니다.**

- 제공 파서는 `none` / `english` / `chinese` / `standard` 뿐 — 한국어 전용 파서 없음.
- `standard`는 **유니코드 문자 분할(Unicode Text Segmentation)** 기반이라 공백·구두점 기준으로만 자름 → **조사/어미를 처리 못 함**.
  - 예: `"검색을 한다"` → `검색을`, `한다`로 잘림 → 사용자가 `검색`으로 찾으면 매칭 안 됨 ❌
  - 한국어는 어절에 조사가 붙어("검색/검색을/검색이/검색에서") 어근 단위 색인이 안 되면 recall이 떨어짐.

### 현실적 선택지
1. **`standard` + 어절 단위로 만족** — 정확한 단어/명사 위주면 그럭저럭. 조사 변형은 취약.
2. **외부 형태소 분석 후 주입 (권장 실전 패턴)** — StarRocks 밖(적재 파이프라인)에서 mecab-ko/Nori/Okt 등으로 형태소 분석 → 어근 단위로 공백 분리한 텍스트를 별도 컬럼에 넣고, StarRocks는 `standard`/`english` 파서로 색인·검색만 담당.
3. **한국어 검색 품질이 핵심이면** — 전문 검색은 Elasticsearch/OpenSearch(Nori) 병행, StarRocks는 벡터+분석 담당.
4. **임베딩(벡터)으로 보완** — 한국어 임베딩 모델은 조사 변형에 덜 민감 → 하이브리드가 한국어에서 특히 유효.

### 2번 패턴 (외부 토큰화 주입) 상세
> 토큰화는 StarRocks 밖에서 미리 하고, 쪼갠 결과를 StarRocks에 주입한 뒤, StarRocks는 그 쪼갠 것만 색인·검색.

```sql
CREATE TABLE documents (
    id                BIGINT NOT NULL,
    content           STRING,          -- 원본 (표시용)
    content_tokenized STRING,          -- ★ 밖에서 형태소분석한 결과 주입
    INDEX idx (content_tokenized) USING GIN("parser" = "standard")
) ENGINE=OLAP DUPLICATE KEY(id) DISTRIBUTED BY HASH(id);
```
```
적재:  "한글 검색을 합니다" → (밖에서 형태소분석) → "한글 검색 하다" → content_tokenized 에 INSERT
검색:  사용자 입력 "검색" → (똑같이 밖에서 토큰화) → WHERE content_tokenized MATCH '검색'
       → 원문이 "검색을"이었어도 어근 "검색"으로 색인돼 매칭됨 ✅
```
**주의 2가지**
1. 원본 컬럼(표시용)과 토큰화 컬럼(검색용)을 **분리**.
2. **적재 때와 검색 때 같은 형태소 분석기**를 써야 함 (규칙 다르면 매칭 실패).
