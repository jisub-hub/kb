---
tags:
  - ai
  - embedding
  - vector
  - similarity
  - hnsw
created: 2026-06-16
---

# Embedding & Vector Search

> [!summary] 한 줄 요약
> 텍스트(또는 이미지·오디오)를 **의미를 담은 고차원 벡터**로 변환하는 기술. 벡터 공간에서 가까운 것이 의미적으로 유사하다. RAG의 핵심 부품이며, 검색·추천·분류·중복 탐지 등 다양한 용도로 쓰인다.

---

## 1. 임베딩이란

```
"안녕하세요"    → [0.23, -0.41, 0.87, ..., 0.12]  ← 768/1536/3072차원 벡터
"Hello"         → [0.25, -0.39, 0.85, ..., 0.14]  ← 의미가 같으면 벡터도 가깝다
"오늘 날씨는?"   → [0.71,  0.33, 0.02, ..., -0.88] ← 의미가 다르면 벡터도 멀다
```

**핵심 특성**:
- 의미가 같으면 벡터 간 거리가 가깝다
- 의미가 반대면 방향이 반대
- 단어·문장·문서 단위 모두 가능
- 동일 임베딩 모델을 질의·문서 모두에 사용해야 함

---

## 2. 유사도 측정

### 코사인 유사도 (Cosine Similarity) ⭐ 표준

```
cos(A, B) = (A · B) / (|A| × |B|)

범위: -1 ~ 1
  1  = 완전 같은 방향 (의미 동일)
  0  = 직교 (무관)
 -1  = 반대 방향 (반의어)
```

```python
import numpy as np

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 단위 벡터화 후엔 내적(dot product)만으로 코사인 유사도 계산 가능
a_norm = a / np.linalg.norm(a)
b_norm = b / np.linalg.norm(b)
similarity = np.dot(a_norm, b_norm)
```

### L2 거리 (유클리드)

```
dist(A, B) = √(Σ(Aᵢ - Bᵢ)²)
```

정규화된 벡터에선 코사인 유사도와 동등. 비정규화 시 크기 차이에 민감.

### 내적 (Dot Product / IP)

OpenAI, Cohere 등 일부 임베딩 모델은 내적으로 유사도 측정. 크기 정보도 포함.

---

## 3. 임베딩 모델

### 문장 임베딩 (Sentence Embeddings)

문장 수준 임베딩의 기초는 Siamese BERT 네트워크를 이용해 의미적으로 유의미한 문장 벡터를 생성하는 Sentence-BERT 연구에서 비롯됐다 [Reimers & Gurevych, 2019].

| 모델 | 차원 | 컨텍스트 | 특징 |
|---|---|---|---|
| `text-embedding-3-small` (OpenAI) | 1536 | 8191 토큰 | 가성비 |
| `text-embedding-3-large` (OpenAI) | 3072 | 8191 토큰 | 고품질 |
| `voyage-3` (Anthropic) | 1024 | 32K | Claude 최적화 |
| **`bge-m3`** (BAAI) | 1024 | 8192 | 한국어 강함, 오픈소스 ⭐ [Chen et al., 2024] |
| `gte-Qwen2-7B-instruct` | 3584 | 131K | 긴 문서, 최신 SOTA |
| `all-MiniLM-L6-v2` | 384 | 512 | 경량, 로컬 추론용 |

```python
# sentence-transformers (로컬)
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-m3")
embeddings = model.encode(["한국어 문장입니다", "이것도 한국어"])
# → shape: (2, 1024)
```

### 임베딩 모델 선택 기준

| 상황 | 권장 |
|---|---|
| 한국어 중심 | bge-m3, gte-Qwen2 |
| 긴 문서(논문, 계약서) | gte-Qwen2-7B (131K 컨텍스트) |
| 빠른 API 구현 | OpenAI text-embedding-3-small |
| 로컬, 저자원 | all-MiniLM-L6-v2 |
| 프로덕션, 비용 최소화 | bge-m3 자체 호스팅 |

---

## 4. 벡터 인덱스 — ANN (Approximate Nearest Neighbor)

수백만~수십억 개 벡터를 정확한 검색(Exact NN) 대신 **근사 최근접 검색**으로 속도 극대화.

### HNSW (Hierarchical Navigable Small World) ⭐ 표준

HNSW는 계층적 근접 그래프를 구축해 O(log n) 수준의 탐색 복잡도를 달성하는 ANN 알고리즘이다 [Malkov & Yashunin, 2020].

```
계층 구조:
  Layer 2 (희소):  A ─────────── H
  Layer 1 (보통):  A ─── C ─── F ─── H
  Layer 0 (밀집):  A─B─C─D─E─F─G─H  (전체 노드)

검색: 상위 레이어부터 내려오며 탐색 → 후보 좁히기 → Layer 0에서 정밀 탐색
```

| 파라미터 | 의미 | 기본값 | 영향 |
|---|---|---|---|
| `M` | 각 노드의 최대 연결 수 [Malkov & Yashunin, 2020] | 16 | 클수록 정확도↑, 메모리↑ |
| `ef_construction` | 인덱싱 시 후보 수 [Malkov & Yashunin, 2020] | 200 | 클수록 품질↑, 빌드↑ |
| `ef_search` | 검색 시 후보 수 [Malkov & Yashunin, 2020] | 50 | 클수록 정확도↑, 속도↓ |

### 인덱스 비교

| 인덱스 | 메모리 | 속도 | 정확도 | 특징 |
|---|---|---|---|---|
| **HNSW** | 높음 | 빠름 | 높음 | 표준, 갱신 지원 |
| IVF_FLAT | 중간 | 빠름 | 중간 | 큰 데이터셋 |
| IVF_PQ | 낮음 | 매우 빠름 | 낮음 | 대규모 압축 |
| Flat (Exact) | 낮음 | 느림 | 100% | 소규모 정확성 필요 |

---

## 5. 벡터 DB

| DB | 내장 DB | 특징 | 적합 상황 |
|---|---|---|---|
| **pgvector** | PostgreSQL 확장 | 기존 PG 인프라 통합, SQL + 벡터 | 중소 규모, 운영 단순 ⭐ |
| **Qdrant** | 전용 | Rust 기반, 고성능, 필터 풍부 | 대규모, 복잡한 메타데이터 필터 |
| **Weaviate** | 전용 | GraphQL, 멀티모달 | 복잡한 쿼리, 멀티모달 |
| **Milvus** | 전용 | 수십억 벡터, 분산 | 극대규모 |
| **Chroma** | 임베디드 | 로컬/테스트용 | 개발, 프로토타이핑 |
| **Redis Vector** | Redis 모듈 | 초저지연 | 캐시와 벡터 DB 통합 |

### pgvector 사용 예시

```sql
-- 확장 설치
CREATE EXTENSION IF NOT EXISTS vector;

-- 테이블 생성
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536),  -- 차원 수 고정
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- HNSW 인덱스
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 유사도 검색 (코사인)
SELECT id, content, 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;

-- 메타데이터 필터 + 벡터 검색 (하이브리드)
SELECT * FROM documents
WHERE metadata->>'category' = 'legal'
  AND (embedding <=> '[...]'::vector) < 0.3  -- 거리 임계값
ORDER BY embedding <=> '[...]'::vector
LIMIT 10;
```

```
pgvector 연산자:
  <=>  코사인 거리 (1 - 코사인 유사도)
  <->  L2 (유클리드) 거리
  <#>  내적 (음수) → ORDER BY <#> ASC = 내적 큰 것 먼저
```

---

## 6. 하이브리드 검색

벡터(의미) + BM25(키워드) 결합으로 두 방법의 단점 보완.

```
벡터 검색: "파이썬으로 소켓 통신 어떻게 해?"
  → 의미적으로 유사한 내용 잘 찾음
  → "socket", "Python" 같은 고유명사는 놓칠 수 있음

BM25: 키워드 정확 매칭
  → "socket" 포함 문서 정확히 찾음
  → 동의어·맥락 이해 못함

[하이브리드]
  벡터 top-50 + BM25 top-50 → RRF(Reciprocal Rank Fusion) 또는 가중 합산 → 최종 top-10
```

```python
# Elasticsearch 하이브리드 (knn + BM25)
body = {
    "query": {"match": {"content": query_text}},  # BM25
    "knn": {"field": "embedding", "query_vector": embedding, "k": 50},
    "rank": {"rrf": {"window_size": 100, "rank_constant": 60}}  # RRF 병합
}
```

---

## 7. 임베딩 업데이트·운영 ⭐

> [!warning] 임베딩 모델 교체 = 전체 재색인
> 동일 텍스트라도 모델이 다르면 벡터 공간이 완전히 다르다.
> → 모델 업그레이드 시 **모든 문서를 재임베딩** 해야 한다.
> → 모델 ID를 메타데이터로 저장해 버전 관리.

### 7.1 왜 신·구 벡터를 섞으면 안 되나

```
모델 A의 벡터와 모델 B의 벡터는 좌표계 자체가 다름
  → 둘을 한 인덱스에 섞으면 코사인 유사도가 무의미(엉뚱한 검색 결과)
  → "일부만 재색인"도 금지: 전부 같은 모델로 통일돼야 함
```

### 7.2 쿼리·문서 모델 버전 일치 강제 (조용한 실패의 핵심) ⚠️

```
문서는 모델 B로 색인됐는데 런타임 쿼리는 모델 A로 임베딩
  → 에러 없이 "그럴듯하지만 틀린" 결과 → 가장 잡기 어려운 RAG 장애
대응: 쿼리 임베딩 경로와 색인 경로가 같은 모델 ID·버전을 쓰도록 코드/설정에서 강제
      인덱스 메타에 model_id를 박고, 불일치 시 부팅 실패(fail-fast)
```

### 7.3 무중단 재색인 (Blue-Green 인덱스)

```
새 모델로 교체 시 운영 중단 없이:
  1) 새 인덱스(green)에 새 모델로 전량 재색인 (백그라운드, 기존 blue는 서비스 유지)
  2) 샘플 쿼리로 green 품질 검증([[Evaluation]]·RAGAS)
  3) 원자적 스위치(alias 전환) → green을 라이브로, blue는 롤백 대비 보관
비용 산정: 문서 수 × 임베딩 호출 단가/시간 — 수백만 문서면 시간·비용 큼(배치·레이트리밋)
증분: 신규/변경 문서만 재색인하되 모델 교체 시엔 전량 필수(7.1)
```

### 7.4 차원·정규화 변경

```
차원(dim) 변경 → 벡터 인덱스 스키마 재생성 필수(pgvector 컬럼 차원 고정)
정규화 방식 변경(L2 normalize 등) → 유사도 척도 영향 → 재색인 + 척도 재확인(§8)
Matryoshka(MRL) 모델: 차원 절단 시에도 같은 모델 내 호환 — 단 절단 길이는 통일
```

### 7.5 임베딩 드리프트 모니터링

```
코퍼스·쿼리 분포가 시간이 지나며 변함(신조어·도메인 변화)
  → 모델은 그대로인데 검색 품질이 서서히 저하(드리프트)
관측: 검색 Recall/Precision 추세·"답 없음"률·재랭킹 점수 분포를 시계열로
      ([[Evaluation]] 온라인 평가, [[RAG]] §5 조용한 실패와 연계)
대응: 임계 이하로 떨어지면 재평가 → 모델 교체/파인튜닝/청킹 재조정 검토
```

---

## 8. 정규화 및 모델별 주의사항

### 단위 벡터 정규화 필요 여부

| 모델 | 정규화 필요 여부 | 권장 유사도 측정 |
|---|---|---|
| OpenAI text-embedding-3-* | ✅ 이미 정규화됨 | 내적 = 코사인 유사도 |
| **bge-m3** | ✅ 이미 정규화됨 | 내적 |
| all-MiniLM-L6-v2 | ✅ 이미 정규화됨 | 코사인 |
| gte-Qwen2-7B-instruct | ❌ 정규화 필요 | 코사인 (정규화 후 내적) |
| voyage-3 | ✅ | 내적 |

```python
import numpy as np

def normalize(v: np.ndarray) -> np.ndarray:
    return v / np.linalg.norm(v)

# 정규화 후 저장 — 검색 시 내적만으로 코사인 유사도 계산 가능 (속도↑)
embedding = model.encode(text)
embedding_normalized = normalize(embedding)
```

### 라이선스 주의

| 모델 | 라이선스 | 상업 사용 |
|---|---|---|
| OpenAI text-embedding-* | API ToS | ✅ (API 사용) |
| voyage-3 | API ToS | ✅ (API 사용) |
| **bge-m3** | MIT | ✅ 자유 |
| all-MiniLM-L6-v2 | Apache 2.0 | ✅ 자유 |
| gte-Qwen2-7B-instruct | Apache 2.0 | ✅ 자유 |
| E5-large-instruct | MIT | ✅ 자유 |

> [!tip] 상업 서비스라면 오픈소스 라이선스(MIT/Apache) 확인 필수.
> API 기반 임베딩 모델은 요청 데이터가 제공사에 전송 → 민감 데이터는 자체 호스팅 검토.

### bge-m3 가변 차원 출력

bge-m3는 `encode()` 파라미터에 따라 dense/sparse/colbert 벡터 모두 출력 가능:

```python
from FlagEmbedding import BGEM3FlagModel

model = BGEM3FlagModel("BAAI/bge-m3", use_fp16=True)

# Dense 벡터 (1024차원) — 일반 검색
output = model.encode(texts, return_dense=True)
dense_vecs = output["dense_vecs"]  # (N, 1024)

# Sparse 벡터 — 키워드 검색 (BM25 대체)
output = model.encode(texts, return_sparse=True)
sparse_vecs = output["lexical_weights"]

# ColBERT 벡터 — 토큰 수준 검색 (고품질, 저장 비용↑)
output = model.encode(texts, return_colbert_vecs=True)
```

---

## 9. 관련
- [[RAG]] · [[LLM]] · [[Transformer-Architecture]]
- [[Chunking-Strategies]] — 임베딩 전 청킹·late chunking(전체 임베딩 후 풀링)
- [[Reranking-Retrieval]] — bi-encoder 검색 후 cross-encoder 리랭킹·RRF 하이브리드 융합
- [[RAG-Ingestion-Pipeline]] — 적재 상류(파싱·증분·삭제 전파): 임베딩의 인입 단계
- [[Self-Query-Retrieval]] — 벡터 DB 네이티브 메타 필터(pre vs post)를 NL에서 자동 추출

---

## 참고문헌

[1] Y. A. Malkov & D. A. Yashunin, "Efficient and Robust Approximate Nearest Neighbor Search Using Hierarchical Navigable Small World Graphs," *IEEE TPAMI*, vol. 42, no. 4, pp. 824–836, 2020. arXiv:1603.09320
[2] N. Reimers & I. Gurevych, "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks," *EMNLP*, 2019. arXiv:1908.10084
[3] J. Chen et al., "BGE M3-Embedding: Multi-Lingual, Multi-Functionality, Multi-Granularity Text Embeddings Through Self-Knowledge Distillation," *ACL Findings*, 2024. arXiv:2402.03216
