---
tags:
  - ai
  - rag
  - reranking
  - retrieval
  - embedding
created: 2026-06-19
---

# 리랭킹 & 2단계 검색·하이브리드 융합

> [!summary] 한 줄 요약
> 벡터 검색(bi-encoder)은 **빠르지만 근사** — 상위 후보를 넉넉히 뽑은 뒤 **cross-encoder 리랭커로 재정렬**하면 정확도가 크게 오른다(2단계 검색). 여러 검색기(BM25+dense)를 섞을 땐 **RRF**로 순위를 융합한다. 청킹 다음으로 검색 정확도 ROI가 큰 레버.

> 청킹은 [[Chunking-Strategies]], 임베딩·하이브리드는 [[Embedding]], 파이프라인은 [[RAG]]. 이 노트는 **검색 후 "재정렬·융합"** 단계.

---

## 1. bi-encoder vs cross-encoder — 왜 2단계인가

```
bi-encoder(임베딩 검색): 쿼리와 문서를 "각각 따로" 벡터로 → 코사인 유사도
  빠름(문서는 미리 임베딩·인덱싱) / 근사(쿼리-문서 상호작용을 못 봄)
cross-encoder(리랭커): 쿼리+문서를 "함께" 모델에 넣어 관련도 점수 직접 산출
  정밀(토큰 수준 상호작용 봄) / 느림(쿼리마다 문서별 추론, 사전 인덱싱 불가)
→ 둘의 장점만: bi로 넓게 후보 → cross로 좁게 정밀 재정렬
```

## 2. 2단계 검색 (retrieve → rerank)

```
1단계 retrieve : bi-encoder로 top-N 빠르게 (N = 50~100, 넉넉히)
2단계 rerank   : cross-encoder로 그 N개를 재채점 → top-k (k = 3~10) 최종
효과: bi가 놓친 "의미는 맞는데 표면이 다른" 후보를 cross가 끌어올림
     (1단계 Recall 확보 → 2단계 Precision 확보)
⚠️ N을 너무 작게 잡으면(예 5) 정답이 1단계에서 이미 탈락 → 리랭커가 구제 못 함
   → N은 넉넉히(Recall), k는 작게(컨텍스트·비용)
```

## 3. 리랭커 모델

```
bge-reranker (BAAI)   — 오픈소스, 다국어, 로컬 구동(한국어 포함)
Cohere Rerank         — API, 강력·간편(유료)
Jina Reranker         — 오픈/API, 긴 문서
mxbai-rerank 등       — 경량 오픈소스
선택: 로컬·프라이버시·다국어 → bge-reranker / 간편·최고 성능 → Cohere
```

## 4. RRF — 여러 검색기 순위 융합

```
Reciprocal Rank Fusion: 점수 스케일이 다른 검색기들을 "순위"만으로 합침
  score(d) = Σ 1 / (k + rank_i(d))     (k는 상수, 보통 60)
용도: 하이브리드 검색 — BM25(키워드) + dense(의미) 순위를 융합
  → 키워드 정확 매칭 + 의미 유사를 둘 다 취함([[Embedding]] 하이브리드)
장점: 점수 정규화 불필요(스케일 달라도 OK)·구현 단순·견고
RRF vs 리랭킹: RRF=여러 "리스트 융합"(저비용) / 리랭킹=cross-encoder "재채점"(고비용·고정밀)
            실무 흔한 조합: BM25+dense → RRF 융합 → cross-encoder 리랭크
```

## 5. 지연·비용 트레이드오프 ⚠️

```
리랭크 비용 = N개 (쿼리,문서) 쌍을 cross-encoder로 추론 → N에 비례
  N=100이면 쿼리당 100회 cross 추론 → 지연·비용↑
균형: N(Recall) ↔ 지연 예산. 보통 N=20~100, k=3~10
완화: 리랭커를 GPU·배치 추론 / 작은 리랭커 모델 / 1단계에서 적당히 필터
원칙: 리랭킹은 "1단계 Recall이 충분할 때" 효과 — 1단계가 정답을 못 뽑으면 무의미
```

## 6. 언제 쓰나

```
거의 항상 추가 이득: 2단계 리랭킹은 RAG 정확도의 표준 향상책(청킹 다음 1순위)
하이브리드면: BM25+dense → RRF 융합은 기본값처럼 권장
스킵 고려: 초저지연 요구 + 1단계만으로 충분한 단순 도메인
측정: Recall@N(1단계)·nDCG/Precision@k(리랭크 후)로 N·k·모델 선택([[Evaluation]])
```

---

## 7. 관련
- [[RAG]] — 검색 파이프라인(§4 품질 기법)
- [[Chunking-Strategies]] — 리랭킹 전(前) 단계: 무엇을 검색 단위로
- [[Embedding]] — bi-encoder 임베딩·하이브리드(BM25+dense)
- [[GraphRAG]] — 관계 기반 검색(리랭킹과 결합 가능)
- [[Evaluation]] — Recall@N·nDCG·Precision@k로 리랭킹 효과 측정
- [[RAG-Latency-Reality]] — 리랭크 지연 vs 정확도 트레이드오프 맥락
- [[Query-Transformation]] — 검색 전 쿼리 개선(multi-query 결과를 RRF로 융합)
- [[Agentic-RAG]] — 검색 품질 평가(CRAG grader)를 능동 게이트로 — 리랭커 점수의 결정화
- [[RAG-Failure-Debugging]] — 순위(②) 실패 진단: gold가 top-N엔 있는데 top-k 밖이면 리랭킹
- [[Context-Assembly-Compression]] — 리랭킹 후 단계: 뽑은 청크의 배치 순서·압축·예산
