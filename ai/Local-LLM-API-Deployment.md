---
tags:
  - ai
  - llm
  - deployment
  - local
  - api
  - agent
  - apple-silicon
created: 2026-06-18
---

# 로컬 LLM API 배포 — Mac 로컬 에이전트 시나리오

> [!summary] 한 줄 요약
> Mac에서 오픈 LLM(Hermes 등)을 띄워 **OpenAI 호환 API**로 추상화하면, 비용 0·프라이버시·오프라인으로 에이전트를 개발할 수 있다. **개발·프로토타입·단일~소수 사용자엔 최적**이지만, **다수 동시 사용자 프로덕션엔 부적합**(Mac은 동시성·HA 약함). 핵심 설계: 처음부터 OpenAI 호환으로 추상화 → 나중에 `base-url` 한 줄로 서버 이전.

---

## 1. Hermes 모델을 써야 하나? — 아니다 (선택)

```
"Hermes 에이전트 방식(function calling)" ≠ "Hermes 모델 필수"

function calling 목적:
  Hermes 3/4 — 포맷(<tool_call>) 내재화, 가장 검증됨 → 안전한 기본값
  대안: Qwen2.5, Llama 3.1+, Mistral 도 tool calling 지원 (Ollama가 템플릿 처리)

Hermes Agent 프레임워크(2026.02):
  provider 자유 → 백엔드 모델 자유 선택 (예: Gemma 4 12B)
```

| 목적 | 권장 모델 | 비고 |
|------|----------|------|
| 도구 호출 신뢰성 최우선 | Hermes 3/4 | 포맷 내재화, 낮은 거부율 → [[Hermes-Agent]] |
| 코딩/수학 강한 에이전트 | Qwen2.5-Coder | tool calling 지원 |
| 범용·최신 | Llama 3.3 70B | 생태계 넓음 |
| 소형·빠름 | Hermes 3 8B / Qwen2.5 7B | 개발·프로토타입 |

> 결론: **Hermes가 안전한 출발점이지만 강제는 아니다.** OpenAI 호환 API로 추상화하면 모델 교체가 쉽다.

---

## 2. 서빙 스택 (Mac 현실)

```
vLLM ❌ — CUDA 전용, Mac 불가
Mac에서 쓰는 것:
  Ollama       — 가장 쉬움. Hermes 템플릿 + tool calling + OpenAI 호환 API 내장
  mlx_lm.server — Apple MLX, OpenAI 호환, Apple Silicon 최적화
  llama.cpp     — Metal 백엔드, GGUF, 세밀 제어
```

```bash
# Ollama — 가장 빠른 시작
ollama pull hermes3:8b
ollama serve                       # OpenAI 호환 API :11434
# → POST http://localhost:11434/v1/chat/completions (tools 지원)
```

→ Mac 추론 원리(대역폭=decode 속도)는 [[Apple-Silicon-Inference]].

---

## 3. 모델 크기 vs Mac 사양

`decode TPS ≈ 메모리 대역폭 / 모델 크기`:

| 모델 | Mac 요건 | 체감 | 용도 |
|------|---------|------|------|
| 8B | 어떤 M-series든 | 매우 빠름(수십~100+ TPS) | 개발·에이전트 기본 |
| 36B | M Max(36GB+) | 쾌적 | 품질·속도 균형 |
| 70B (INT4) | M Max 64GB+ / Ultra | 느림(~17 TPS @M5 Max) | 품질 우선·소수 사용자 |

> 에이전트 개발은 **8B로 시작**이 현실적. function calling은 8B도 충분.

---

## 4. OpenAI 호환 API로 개발

```python
# 기존 OpenAI SDK 그대로, base_url만 로컬로
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

resp = client.chat.completions.create(
    model="hermes3:8b",
    messages=[{"role":"user","content":"서울 날씨 알려줘"}],
    tools=[{"type":"function","function":{
        "name":"get_weather",
        "parameters":{"type":"object",
            "properties":{"city":{"type":"string"}},"required":["city"]}}}],
)
# resp.choices[0].message.tool_calls → 실행 → 결과를 messages에 추가 → 재호출(루프)
```

```java
// Spring AI — base-url만 로컬 Ollama로
spring:
  ai:
    openai:
      base-url: http://localhost:11434
      chat: { options: { model: hermes3:8b } }
```

에이전트 루프·도구 설계는 [[Agent-Harness]], 도구 호출 포맷은 [[Hermes-Agent]].

---

## 5. 장점 / 한계 (정직하게)

| 장점 ✅ | 한계 ⚠️ |
|--------|--------|
| API 비용 0 (무제한 호출) | **동시성**: continuous batching 없음 → 다수 사용자 시 큐잉·TPS 급락 |
| 프라이버시 (데이터 로컬) | **운영**: Mac은 24/7 서버 아님 (HA·자동재시작·모니터링 부재) |
| 오프라인 동작 | **추론 깊이**: 오픈모델 < Claude/GPT (복잡 멀티스텝 갭) |
| function calling 내재화(Hermes) | **속도**: 70B는 대역폭 한계로 느림 |
| 개발 반복 빠름 | function calling JSON 파싱 실패 대비 필요 |

→ 동시성·capacity 상세는 [[LLM-System-Performance-Analysis]], 서버 GPU 선택은 [[GPU-Inference-Decision-Guide]].

---

## 6. 적합 / 부적합

```
적합 ✅
  개인 개발·프로토타입 / 프라이버시 민감(로컬) / 오프라인·엣지
  비용 0 실험 / 사내 소수 사용자 도구

부적합 ❌
  다수 동시 사용자 프로덕션 / 저지연 대규모 트래픽
  24/7 가용성 SLA / 최고 추론 품질 요구
```

---

## 7. 권장 아키텍처 — 개발→프로덕션 이전 경로

```
[개발] Mac 최적
  Mac + Ollama + Hermes 3 8B
    → OpenAI 호환 API (localhost:11434)
    → 앱 백엔드(Spring AI / FastAPI) function calling 루프
    → 비용 0·프라이버시로 에이전트 로직 빠르게 검증

[프로덕션 전환] 다수 사용자면
  OpenAI 호환 인터페이스 유지 → base-url 교체만:
    → NVIDIA + vLLM (continuous batching) → [[vLLM]] / [[GPU-Inference-Decision-Guide]]
    → 또는 클라우드 API
  ※ 앱 코드는 그대로, 백엔드만 교체
```

> 핵심 설계 원칙: **처음부터 OpenAI 호환 API로 추상화**하라. 그러면 "Mac 개발 → 서버 배포"가 `base-url` 한 줄 교체로 끝난다. 백엔드가 Mac이든 클라우드든 앱은 무관.

---

## 8. 정리

```
Mac 로컬 Hermes API = 개발·프로토타입·단일사용자·프라이버시에 최적
모델: Hermes 권장(필수 아님) — Qwen/Llama도 가능, OpenAI 호환으로 교체 자유
스택: Ollama(가장 쉬움) / MLX / llama.cpp — vLLM 불가
함정: 다수 사용자 프로덕션엔 동시성·HA 부족 → vLLM/NVIDIA로 이전
설계: OpenAI 호환 추상화 → 이전이 base-url 한 줄
```

---

## 관련
- [[Hermes-Agent]] — function calling 포맷·에이전트 루프
- [[Apple-Silicon-Inference]] — Mac 추론 원리(대역폭=속도)
- [[Open-Source-Models]] — 모델 선택 (Hermes/Qwen/Llama)
- [[Agent-Harness]] — 에이전트 루프·도구 설계
- [[vLLM]] · [[GPU-Inference-Decision-Guide]] · [[LLM-System-Performance-Analysis]] — 프로덕션 이전
- [[RAG-Chatbot-E2E]] — Spring AI + 로컬/원격 LLM 연동 예시
- [[Hermes-Deployment-Surface]] — raw 모델이 아닌 *에이전트 전체*를 API/게이트웨이로 노출
