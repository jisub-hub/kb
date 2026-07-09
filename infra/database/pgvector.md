---
tags:
  - database
  - postgresql
  - vector-db
  - ai
  - rag
created: 2026-06-16
---

# pgvector — PostgreSQL 벡터 확장

> [!summary] 한 줄 요약
> PostgreSQL 확장으로 벡터를 저장·검색한다. 별도 벡터 DB 없이 **기존 PG 인프라에 통합**할 수 있어 중소 규모 [[../../ai/RAG|RAG]] 파이프라인에서 사실상 표준.

---

## 1. 설치

```sql
-- PostgreSQL 15+ 권장
CREATE EXTENSION IF NOT EXISTS vector;
```

```bash
# Docker
docker run -e POSTGRES_PASSWORD=pass -p 5432:5432 pgvector/pgvector:pg16

# AWS RDS: pg_vector 파라미터 그룹 활성화 후 콘솔에서 Extension 허용
# Homebrew (로컬)
brew install pgvector
```

---

## 2. 테이블 정의 & 인덱스

```sql
-- 벡터 컬럼 선언 (차원 수는 임베딩 모델에 맞춰야 함)
-- bge-m3: 1024, text-embedding-3-small: 1536, text-embedding-ada-002: 1536
CREATE TABLE documents (
    id          BIGSERIAL PRIMARY KEY,
    content     TEXT        NOT NULL,
    source      TEXT,                          -- 출처 메타데이터
    section     TEXT,
    created_at  TIMESTAMPTZ DEFAULT now(),
    embedding   vector(1024)                   -- 차원 수 고정
);

-- ─────────────────────────────────────────────────────────
-- 인덱스 선택: IVFFlat vs HNSW
-- ─────────────────────────────────────────────────────────

-- IVFFlat: 빌드 빠름, 정확도 약간 낮음, 소규모(< 100만) 적합
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);   -- lists ≈ sqrt(행 수) 기준 시작

-- HNSW: 빌드 느림(메모리↑), 검색 빠르고 정확, 대규모 권장 ★
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
-- m: 각 노드의 최대 연결 수 (16~64). 높을수록 정확도↑, 메모리↑
-- ef_construction: 빌드 시 탐색 깊이 (64~200). 높을수록 품질↑, 빌드 느림
```

### 거리 함수 선택

| 연산자 | 거리 | 인덱스 ops | 언제 |
|--------|------|-----------|------|
| `<->` | L2 (유클리디안) | `vector_l2_ops` | 정규화 안 된 벡터 |
| `<=>` | 코사인 유사도 | `vector_cosine_ops` | 텍스트 임베딩 **표준** ⭐ |
| `<#>` | 내적(음수) | `vector_ip_ops` | dot-product로 학습된 모델 |

---

## 3. 검색 쿼리

```sql
-- 코사인 유사도 기반 top-5 검색
SELECT id, content, source,
       1 - (embedding <=> '[0.12, 0.34, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.12, 0.34, ...]'::vector
LIMIT 5;

-- 메타데이터 필터링 + 벡터 검색 (하이브리드)
SELECT id, content, source,
       embedding <=> $1::vector AS distance
FROM documents
WHERE section = 'finance'                  -- 사전 필터 → 검색 범위 축소
  AND created_at > now() - interval '1 year'
ORDER BY distance
LIMIT 10;

-- HNSW 검색 정확도 튜닝 (런타임 파라미터)
SET hnsw.ef_search = 100;   -- 기본 40, 높일수록 정확도↑ 속도↓
```

---

## 4. Spring AI 연동

```groovy
implementation 'org.springframework.ai:spring-ai-pgvector-store'
```

```yaml
spring:
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW          # HNSW | IVFFLAT
        distance-type: COSINE_DISTANCE
        dimensions: 1024          # 임베딩 모델 차원과 일치
        initialize-schema: true   # 첫 실행 시 테이블 자동 생성
```

```java
@Service
@RequiredArgsConstructor
public class DocumentIndexer {
    private final VectorStore vectorStore;   // PgVectorStore 빈 주입

    public void index(List<Document> docs) {
        // TokenTextSplitter: 청킹 + 임베딩 + 저장을 한 번에
        var chunks = new TokenTextSplitter(800, 200, 5, 10000, true).apply(docs);
        vectorStore.add(chunks);
    }

    public List<Document> search(String query, String section) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(10)
                .withSimilarityThreshold(0.7)           // 유사도 임계값 (0~1)
                .withFilterExpression("section == '" + section + "'")
        );
    }
}
```

---

## 5. 성능 튜닝

```sql
-- 인덱스 빌드 시 메모리 임시 확장
SET maintenance_work_mem = '2GB';   -- HNSW 빌드 시 필요
CREATE INDEX ... USING hnsw ...;
RESET maintenance_work_mem;

-- 병렬 빌드
SET max_parallel_workers = 4;
CREATE INDEX CONCURRENTLY ...;      -- 운영 중 온라인 인덱스 빌드

-- 통계 확인
SELECT pg_size_pretty(pg_relation_size('documents'));
SELECT pg_size_pretty(pg_indexes_size('documents'));
\d+ documents   -- 인덱스 목록 확인
```

### 규모별 인덱스 선택 가이드

| 행 수 | 권장 | 파라미터 기준 |
|-------|------|--------------|
| < 10만 | 인덱스 없이 순차 스캔도 충분 | — |
| 10만 ~ 100만 | IVFFlat | lists = sqrt(N) |
| 100만 ~ | HNSW | m=16~32, ef_construction=64~128 |

---

## 6. 하이브리드 검색 (벡터 + 키워드)

```sql
-- BM25 키워드 + 벡터 결합 (RRF: Reciprocal Rank Fusion)
WITH vector_results AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <=> $1) AS rank
    FROM documents
    ORDER BY embedding <=> $1 LIMIT 50
),
keyword_results AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY ts_rank(to_tsvector('korean', content), query) DESC) AS rank
    FROM documents, to_tsquery('korean', $2) query
    WHERE to_tsvector('korean', content) @@ query
    LIMIT 50
)
SELECT COALESCE(v.id, k.id) AS id,
       1.0/( 60 + COALESCE(v.rank, 1000)) + 1.0/(60 + COALESCE(k.rank, 1000)) AS rrf_score
FROM vector_results v
FULL OUTER JOIN keyword_results k USING (id)
ORDER BY rrf_score DESC
LIMIT 10;
```

---

## 7. 관련
- [[../../ai/RAG]] · [[PostgreSQL-Internals]] · [[Connection-Pool]]
- [[../../programming/spring/data/RDS-vs-Self-Hosted]]
