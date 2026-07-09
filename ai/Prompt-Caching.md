---
tags:
  - ai
  - llm
  - prompt-caching
  - cost-optimization
  - inference
  - latency
created: 2026-06-18
---

# 프롬프트 캐싱 (Prompt Caching)

> [!summary] 한 줄 요약
> 프롬프트의 **반복되는 앞부분(system prompt·문서·도구 정의)의 prefill 결과(KV cache)를 저장해 재사용**하는 기법. 효과는 **TTFT 단축 + 입력 비용 ~90% 절감** 두 가지. 핵심 원칙은 **고정 내용을 앞에, 변하는 질문을 뒤에.** 답변(출력)은 캐싱하지 않으므로 매번 새로 생성된다.

---

## 1. 무엇을 캐싱하나 — 입력의 prefill 결과

LLM은 입력 토큰 전부의 K·V(attention 상태)를 계산(prefill)한 뒤 토큰을 생성한다. 같은 앞부분을 매 요청 다시 계산하는 건 낭비다.

```
일반: [긴 system prompt + 문서 2000토큰] + [질문]  → 매번 2000토큰 전부 prefill
캐싱: 앞 2000토큰의 KV를 저장 → 다음 요청에서 그 부분 prefill 스킵
```

→ KV·prefill 배경은 [[CUDA-Role-in-LLM]] · [[LLM-System-Performance-Analysis]](TTFT).

---

## 2. ⚠️ 프롬프트 캐싱 ≠ 응답 캐싱 (가장 흔한 오해)

```
프롬프트 캐싱 (이 노트)
  = 입력 "앞부분(KV)" 재사용 → 출력은 매번 새로 생성
  → "맘에 안 드는 답이 박제되어 계속 나온다" = 발생 안 함

응답/시맨틱 캐싱 (Semantic Cache)
  = 같은 질문 → 저장된 "답"을 그대로 반환
  → 부적절한 답이 고정될 수 있음 → TTL·무효화로 관리
  → [[RAG-Latency-Reality]] 5절
```

비유: 프롬프트 캐싱 = "교과서를 매번 다시 안 읽고 기억"(답은 새로 작성) / 응답 캐싱 = "답안지를 통째로 저장해 재제출"(틀리면 계속 틀림).

---

## 3. 동작 원리 — Prefix 매칭

```
요청1: [system+문서 (고정)] + [질문A]  → 고정부 KV 저장, 답A 생성
요청2: [system+문서 (고정)] + [질문B]  → 고정부 캐시 히트(prefill 스킵), 답B 새로 생성
        ↑ 앞부분이 "정확히 일치"해야 히트 (1토큰만 달라도 그 이후 전부 무효)
```

> 그래서 **고정 내용을 앞에, 변하는 내용을 뒤에** 배치해야 한다. 변하는 게 앞에 오면 매번 캐시 미스.

```
✅ [고정 system][고정 문서][가변 질문]
❌ [가변 질문][문서]  ← 앞이 매번 바뀜 → 캐시 무용
```

Prompt Cache [Gim et al., 2023]는 재사용 가능한 세그먼트의 attention 상태를 사전 계산·저장해 GPU 최대 8×, CPU 최대 60× 속도 향상을 보였다.

---

## 4. 두 레벨 — 서빙 엔진 vs API

| 레벨 | 방식 | 비고 |
|------|------|------|
| **서빙 엔진** | vLLM `--enable-prefix-caching` | 자체 호스팅, 동일 prefix 요청 간 KV 블록 공유(여러 사용자 공유 가능) → [[vLLM]] |
| **API** | Anthropic `cache_control` / OpenAI 자동 | 클라우드, 명시 표시(Anthropic) 또는 자동 감지(OpenAI) |

```python
# Anthropic — 캐싱할 구간 명시
response = anthropic.messages.create(
    model="claude-sonnet-4-6",
    system=[{
        "type": "text",
        "text": entire_knowledge_base,        # 고정 컨텍스트
        "cache_control": {"type": "ephemeral"} # ← 이 구간 캐싱
    }],
    messages=[{"role": "user", "content": question}],  # 가변
)
# usage.cache_read_input_tokens 로 캐시 적중 확인
```

```bash
# vLLM 로컬
vllm serve <model> --enable-prefix-caching
```

---

## 5. 효과 — 두 가지

```
① TTFT 단축: 앞부분 prefill 스킵 → 첫 토큰까지 빨라짐 (긴 컨텍스트일수록 큼)
② 비용 절감: 캐시된 입력 토큰 ~90% 싸게 과금 (API)
```

비용 정책 (Anthropic 예시, 변동성 있음 → 공식 확인):
```
캐시 쓰기: ~1.25× (5분 TTL) / ~2× (1시간 TTL)  ← 첫 요청 약간 비쌈
캐시 읽기: ~0.1× (90% 할인)                      ← 재사용 시 대폭 저렴
→ 재사용이 충분히 많아야 이득 (손익분기: 보통 2회 이상 히트)
```

---

## 6. 효과가 큰 상황 vs 작은 상황

| 상황 | 효과 |
|------|------|
| 긴 고정 system prompt 반복 | ⭐⭐⭐ |
| 같은 문서로 여러 질문 (문서 QA) | ⭐⭐⭐ |
| 에이전트 (도구 정의·시스템 반복, 멀티턴) | ⭐⭐⭐ (비용 주범이 누적 input → 캐싱 결정적) |
| few-shot 예시 고정 | ⭐⭐ |
| 매번 다른 짧은 질문만 | ✕ (캐시할 고정부 없음) |

---

## 7. 주의점 (캐시 미스·함정)

```
- prefix 1토큰이라도 다르면 그 이후 전부 캐시 미스 (고정부에 타임스탬프·랜덤 금지)
- TTL 만료: 뜸한 요청은 캐시 만료 후 재계산 (API 보통 5분~1시간)
- 최소 캐시 길이: 너무 짧으면 캐싱 안 됨 (모델별 ~1024~4096 토큰)
- 캐시 쓰기 비용: 첫 요청은 약간 비쌈 → 1회성 요청엔 오히려 손해
- 동적 콘텐츠는 뒤로: 변하는 부분(질문·시각)을 prefix에 넣지 말 것
```

---

## 8. CAG와의 관계

문서가 **완전히 고정**이고 반복 질의되면, 아예 미리 KV를 계산해두는 **CAG(Cache-Augmented Generation)**가 RAG의 대안이 된다 — 검색 없이 캐시된 전체 문서에 바로 질문. → [[RAG-vs-Knowledge-Systems]]

---

## 9. 비용 계산 예시

```
문서 QA: 고정 문서 5000토큰 + 질문 100토큰, 하루 1000회 질의

캐싱 없음: 5100토큰 × input단가 × 1000회
캐싱 있음: 첫 요청만 5000토큰 풀 + 쓰기, 이후 999회는 5000토큰을 ~0.1×로
  → 입력 비용 ~85~90% 절감
  + TTFT도 단축 (5000토큰 prefill 스킵)
```

→ 비용 레버 전체는 [[LLMOps]] 3절.

---

## 10. 정리

```
프롬프트 캐싱 = 입력 앞부분(KV) 재사용 → TTFT↓ + 입력비용 ~90%↓
원칙: 고정 앞 · 가변 뒤 (prefix 정확 일치해야 히트)
레벨: vLLM --enable-prefix-caching / API cache_control(자동)
효과 큼: 반복 system prompt·문서 QA·에이전트 / 효과 없음: 매번 다른 짧은 질문
주의: 1토큰 차이 미스·TTL·최소길이·쓰기비용
혼동 금지: 응답(답)을 저장하는 Semantic Cache와 다름 (답은 매번 새로)
```

---

## 참고문헌
[1] I. Gim et al., "Prompt Cache: Modular Attention Reuse for Low-Latency Inference," 2023. arXiv:2311.04934
[2] Anthropic Prompt Caching / OpenAI Prompt Caching 공식 문서

---

## 관련
- [[Prompt-Engineering]] — 프롬프트 설계(캐싱 맥락)
- [[RAG-Latency-Reality]] — Semantic Cache(응답 캐싱, 구분)
- [[Inference-Optimization]] · [[vLLM]] — 서빙 레벨 prefix caching
- [[LLM-System-Performance-Analysis]] — TTFT/prefill, capacity
- [[RAG-vs-Knowledge-Systems]] — CAG
- [[LLMOps]] — 비용 레버
- [[Tool-Search]] — 입력 비용 절감의 보완책(캐싱=재사용 vs 툴서치=미노출)
- [[Chunking-Strategies]] — Contextual Retrieval 인덱싱이 문서 캐시로 청크 맥락 생성 비용↓
- [[CAG-Cache-Augmented-Generation]] — 지식 전체를 KV 캐시에 고정해 검색 없이 답(이 메커니즘의 RAG 적용)
