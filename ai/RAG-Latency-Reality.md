---
tags:
  - ai
  - rag
  - latency
  - performance
  - architecture
  - deep-dive
created: 2026-06-16
---

# "RAG는 느리다" — 무엇이 진짜 병목인가

> [!insight] 핵심 주장
> "RAG는 느리다"는 말은 절반만 맞다. **느린 것은 검색이 아니라 LLM 생성이다.**
> 따라서 wiki를 아무리 잘 구조화해도 근본 병목은 사라지지 않는다.
> "결국 같지 아니한가"는 **병목 관점에서는 맞다.**

---

## 1. RAG 지연 시간 실측 분해

실제 RAG 파이프라인의 각 단계별 지연을 측정하면:

```
[일반적인 RAG 레이턴시 분해]

단계                        시간       전체 비중
────────────────────────────────────────────────
① 쿼리 임베딩               5~20ms      0.3%
② 벡터 검색 (HNSW, 10만건)  2~10ms      0.2%
③ 청크 조회 + 프롬프트 조립  5~20ms      0.5%
④ LLM 생성 (500토큰 출력)   1000~4000ms 98%+
────────────────────────────────────────────────
합계                        ~1~4초

* Claude Sonnet 기준, 스트리밍 없을 때
```

**검색 전체 = 12~50ms. LLM 생성 = 1000~4000ms.**

"RAG가 느리다"고 할 때 사람들이 체감하는 그 느림은 전부 ④번이다.
①②③을 10배 최적화해도 총 지연이 2% 줄어든다.

### ④번을 더 쪼개면 — TTFT(prefill) + 생성시간(decode)

LLM 생성(④)은 다시 두 구간으로 나뉘고, **병목 요인이 다르다.** RAG는 검색된 청크가 컨텍스트로 들어가 **입력이 길기 때문에 prefill 비중이 크다.**

```
사용자 체감 시간 (end-to-end)
│
├─ TTFT (첫 토큰까지) ─── prefill ─── compute-bound (FLOPS)
│    = (검색 청크 + 질문) 입력 토큰 수 × 모델크기 / FLOPS
│    RAG는 컨텍스트가 길어 이 구간이 길다 → "한참 멈춰있다 답 시작"
│
└─ 생성시간 ─── decode ─── memory-bound (대역폭)
     = 출력 토큰 수 × (모델크기 / 대역폭)

총 시간 = TTFT + (출력 토큰 수 × ITL)
```

| RAG 상황 | 지배 구간 | 결정 요인 | 대응 |
|---------|----------|----------|------|
| 짧은 컨텍스트 (Top-3, 짧은 청크) | 생성시간(decode) | 대역폭 | 작은 모델·양자화 |
| **긴 컨텍스트 (Top-10, 큰 청크, 멀티홉)** | **TTFT(prefill)** | **FLOPS** | **prefix caching**, 컨텍스트 축소, FLOPS↑ GPU |

> 그래서 **RAG에서 컨텍스트를 키울수록 TTFT가 늘어난다** — "검색 결과를 많이 넣으면 정확하지만 첫 글자가 늦게 나온다"는 트레이드오프. 고정 system prompt·자주 쓰는 청크는 **prefix caching**으로 prefill을 스킵해 TTFT를 줄일 수 있다. (→ 5절 CAG/프롬프트 캐싱)

---

## 2. 그렇다면 wiki 구조화가 속도에 미치는 영향

```
[구조화된 wiki의 검색 최적화 효과]

A. Top-K를 줄임 (Top-10 → Top-3)
   → 컨텍스트 토큰 수 감소
   → LLM 입력 프롬프트 짧아짐
   → 생성 시간 소폭 감소 (~5~15%)
   
B. 계층적 검색 (어떤 폴더? → 그 안에서 어떤 문서?)
   → 벡터 검색 범위 축소
   → 검색 시간 감소: 10ms → 3ms (절약: 7ms)
   → 전체 지연 개선: 0.3%
   
C. 사전 계산된 요약 노드 (GraphRAG 커뮤니티 요약)
   → 자주 묻는 글로벌 질문은 요약에서 바로 답변
   → 실제로 의미 있는 개선 가능
   → 단, 해당 쿼리 패턴일 때만
```

**결론**: wiki 구조화는 검색 품질(정확도)을 높이고, 컨텍스트 토큰을 줄여 **간접적으로** 속도에 기여하지만, 근본 병목(LLM 생성)은 그대로다.

---

## 3. 그럼 "RAG는 느리다"는 말은 어디서 나왔는가

세 가지 실제 상황이 있다.

### 상황 1 — Multi-hop RAG (진짜 느림)

```
단순 질문: "HNSW란 무엇인가?"
→ 검색 1회 → 생성 1회 → 끝

복잡한 질문: "K8s에서 pgvector를 쓰는 RAG 시스템의 장애 시나리오를 분석해줘"
→ 검색 1: pgvector 관련 문서
→ LLM 판단: "K8s StatefulSet 문제도 봐야겠다"
→ 검색 2: StatefulSet-DB 문서
→ LLM 판단: "HA Cluster 패턴도 필요하다"
→ 검색 3: PostgreSQL-HA-Cluster 문서
→ 최종 생성

총 지연 = LLM 생성 × 3회 + 검색 × 3회
       = ~3~12초 (순차 실행)
```

이 경우 구조화된 wiki는 실제로 빠르게 만들 수 있다.
LLM이 "어느 문서를 볼지" 판단하는 시간을 wikilink 그래프 탐색으로 대체하면 LLM 호출 횟수 자체를 줄인다.

### 상황 2 — Re-ranking 포함 파이프라인

```
Naive RAG:      검색 → 생성             ~1.5초
+Re-ranking:    검색 → 재랭킹 → 생성    ~2초
+HyDE:          가상답변생성 → 검색 → 생성  ~3초
+Self-RAG:      생성 중 재검색 루프      ~5초+
```

"Advanced RAG가 느리다"는 말이 더 정확하다.
정확도를 높이는 기법들이 LLM 호출 횟수를 늘리기 때문이다.

### 상황 3 — 체감 속도 (Streaming 미적용)

```
스트리밍 없음: 4초 대기 → 전체 답변 한 번에 표시
스트리밍 있음: 첫 토큰 0.3초 → 토큰이 실시간으로 표시됨

→ 실제 총 생성 시간은 동일하지만
→ 사용자 체감은 스트리밍이 10배 빠르게 느껴짐
```

많은 경우 "RAG가 느리다"는 불만은 스트리밍이 없어서다.

---

## 4. "결국 같지 아니한가" — 어디서 맞고 어디서 다른가

```
[병목 관점에서]

                     Naive RAG    구조화된 Wiki RAG
검색 지연              12~50ms         5~20ms
LLM 생성 지연          1~4초           1~4초  ← 동일
────────────────────────────────────────────────────
총 지연                ~1~4초          ~1~4초  ← 거의 같음

→ "결국 같다"는 관찰: 맞다.
  LLM을 쓰는 한 이 병목은 바뀌지 않는다.
```

```
[정확도 관점에서]

                     Naive RAG    구조화된 Wiki RAG
엉뚱한 청크 혼입           높음            낮음
맥락 손실                  높음            낮음
글로벌 질문 답변 품질        낮음            높음
────────────────────────────────────────────────────
→ 속도는 같지만 정확도가 다르다.
  구조화의 이득은 "빠름"이 아니라 "정확함"이다.
```

---

## 5. 실제로 속도를 높이는 방법들

wiki 구조화가 아니라 이쪽이 실질적인 속도 개선이다.

### 방법 1 — Semantic Cache (가장 효과적)

> [!important] 응답 캐싱 ≠ 프롬프트 캐싱
> Semantic Cache는 **답변(응답)을 저장**해 같은/유사 질문에 그대로 반환한다 → 캐시된 답이 마음에 안 들면 **계속 같은 답이 나온다.** 이는 **TTL·수동 무효화·피드백 기반 재생성**으로 관리한다(아래). 반면 입력 앞부분만 재사용하고 답은 매번 새로 만드는 것은 **프롬프트 캐싱**이다(답 박제 없음) → [[Prompt-Engineering]] 6절.

```python
# 동일하거나 유사한 질문이 반복되면 캐시에서 즉시 반환
# Redis + 임베딩 유사도로 구현
# ※ 캐시된 답이 부적절하면: TTL 만료 / 피드백 시 무효화 / "다시 생성" 우회

class SemanticCache:
    def __init__(self, redis, threshold=0.95):
        self.redis = redis
        self.threshold = threshold  # 유사도 95% 이상이면 캐시 히트
    
    def get(self, query_embedding):
        # Redis에서 저장된 쿼리 임베딩들과 비교
        cached = self.redis.search_similar(query_embedding, top_k=1)
        if cached and cached.score >= self.threshold:
            return cached.answer  # LLM 호출 없이 즉시 반환 (~5ms)
        return None
    
    def set(self, query_embedding, answer):
        self.redis.store(query_embedding, answer, ttl=3600)

# 효과:
# 캐시 히트: 4000ms → 5ms  (800배 빠름)
# 캐시 미스: 변화 없음
# 반복 질문이 많은 도메인에서 효과적 (FAQ, 지식베이스)
```

### 방법 2 — 스트리밍 (체감 속도)

```java
// Spring AI SSE 스트리밍
@GetMapping(value = "/ask", produces = TEXT_EVENT_STREAM_VALUE)
public Flux<String> askStream(@RequestParam String question) {
    return chatClient.prompt()
        .user(question)
        .stream()
        .content();  // 첫 토큰 0.3초 후 즉시 시작
}

// 실제 생성 시간은 같지만
// 사용자는 첫 글자를 0.3초에 보게 됨
// "느리다" 불만의 70%는 이것으로 해결됨
```

### 방법 3 — 작은 모델 라우팅

```
질문 유형 분류 (fast, cheap 분류기로):
  단순 사실 질문  → GPT-4o-mini / Claude Haiku  (~0.5초)
  복잡한 분석    → GPT-4o / Claude Sonnet      (~2초)
  전문가 수준    → Claude Opus / GPT-4          (~4초)

→ 70%의 질문이 단순 → 70%의 응답이 8배 빠름
→ 전체 평균 레이턴시 대폭 감소
```

### 방법 4 — CAG (KV Cache 사전 계산)

```python
# 문서가 고정되어 있고 자주 질의되는 경우
# 문서를 미리 KV Cache로 처리해두면
# 매 요청마다 문서 토큰을 다시 처리하지 않아도 됨

# Anthropic 프롬프트 캐싱 예시
response = anthropic.messages.create(
    model="claude-sonnet-4-6",
    system=[{
        "type": "text",
        "text": entire_knowledge_base,  # 전체 문서
        "cache_control": {"type": "ephemeral"}  # KV Cache에 저장
    }],
    messages=[{"role": "user", "content": question}]
)

# 첫 요청: 전체 토큰 비용 + 캐시 저장 비용
# 이후 요청: 캐시 히트 → 입력 토큰 비용 90% 절감 + 속도 향상
# (캐시 유지: 5분, 연장 가능)
```

### 방법 5 — 병렬 검색

```python
import asyncio

async def parallel_retrieve(queries: list[str]) -> list[Document]:
    # 여러 하위 쿼리를 동시에 검색 (순차 대신 병렬)
    tasks = [vector_store.asimilarity_search(q) for q in queries]
    results = await asyncio.gather(*tasks)
    return merge_and_deduplicate(results)

# Multi-hop을 병렬화:
# 순차: 검색A(100ms) → 검색B(100ms) → 검색C(100ms) = 300ms
# 병렬: 검색A+B+C 동시 = 100ms
```

---

## 6. 속도 개선 효과 비교

```
방법                    적용 전    적용 후    개선율
────────────────────────────────────────────────────────
wiki 구조화              4.0초      3.8초      5%    ← 미미
Semantic Cache (히트 시)  4.0초      0.005초    99%   ← 압도적
스트리밍                  체감 느림  체감 빠름  주관적
작은 모델 라우팅           3.0초      0.5초      83%   ← 효과적
프롬프트 캐싱 (CAG)        4.0초      1.5초      63%   ← 유효
병렬 Multi-hop            8.0초      3.0초      63%   ← 유효
────────────────────────────────────────────────────────
```

wiki 구조화는 속도 기여가 가장 작다. 그러나 정확도 기여는 가장 크다.

---

## 7. 최종 프레임 — 왜 이 논쟁이 잘못 세팅되어 있는가

"RAG는 느리다 → wiki를 잘 만들면 빨라지나?" 라는 질문은 전제가 틀려있다.

```
잘못된 프레임:
  RAG 속도 문제 → 지식 구조화로 해결

올바른 프레임:
  LLM 생성 속도 문제 → 캐싱/스트리밍/모델선택으로 해결
  RAG 정확도 문제    → 지식 구조화로 해결
```

두 문제는 서로 다른 문제이고, 서로 다른 해결책이 있다.

**"결국 같지 아니한가"** 에 대한 최종 답:

```
속도 측면에서:  맞다. LLM을 쓰는 이상 구조화는 병목을 바꾸지 않는다.
정확도 측면에서: 다르다. 구조화된 wiki는 훨씬 정확한 컨텍스트를 제공한다.

→ wiki 구조화의 가치는 "빠름"이 아니라 "덜 틀림"이다.
→ 느림을 해결하려면 캐싱을 해야 한다.
→ 그리고 잘 구조화된 wiki는 캐싱 효과가 더 크다:
   예측 가능한 질문 패턴 → 캐시 히트율 높음
   = 구조화의 간접 속도 효과
```

---

## 8. 관련
- [[RAG-vs-Knowledge-Systems]] — "LLM Wiki도 RAG인가?" 이전 논의
- [[RAG]] — 청킹, 임베딩, 리랭킹 기술
- [[LLMOps]] — 비용·속도 최적화, 프롬프트 캐싱
- [[Inference-Optimization]] — LLM 생성 자체를 빠르게 (KV Cache, 양자화, vLLM)
- [[Prompt-Engineering]] — 프롬프트 캐싱 (Anthropic)
- [[RAG-Failure-Debugging]] — 같은 파이프라인을 "정확도"로 분해(이 노트는 "속도")
