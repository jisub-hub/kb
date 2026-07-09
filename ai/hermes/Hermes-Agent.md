---
tags:
  - ai
  - llm
  - agent
  - hermes
  - function-calling
  - open-source
created: 2026-06-18
---

# Hermes 에이전트 — Function Calling & 자율 에이전트

> [!summary] 한 줄 요약
> NousResearch의 **Hermes 모델**(3/4/4.3)은 `<tools>`/`<tool_call>`/`<tool_response>` XML 포맷을 파인튜닝으로 내재화해 도구 호출 에이전트에 강하다. 시스템 프롬프트 준수·낮은 거부율·구조화 출력이 강점이라 **로컬 자율 에이전트**의 사실상 표준. 2026.02부터 동명의 **Hermes Agent 프레임워크**(자가개선·지속 메모리)도 별도 존재한다.

---

## 0. ⚠️ 두 개의 "Hermes" 구분

```
① Hermes 모델 (LLM)        — NousResearch 파인튜닝 LLM (Hermes 3/4/4.3)
② Hermes Agent (프레임워크) — 2026.02 출시 오픈소스 자율 에이전트 (MIT, 46k+ stars)

"hermes agent"는 보통 ①을 function calling 에이전트로 쓰는 것을 뜻하지만,
2026.02부터 ②라는 별도 에이전트 프레임워크 제품도 존재한다.
```

---

## 1. Hermes 모델 계보

| 버전 | 출시 | 기반 | 특징 |
|------|------|------|------|
| Hermes 3 | 2024.08 | Llama 3.1 (8B/70B/405B) | agentic·function calling 본격화 |
| Hermes 4 | 14B(2025.01), 70B/405B FP8(2025.09) | Llama 계열 | `<tool_call>` 포맷 성숙, 거부율↓ |
| **Hermes 4.3** | 2025.08 | ByteDance Seed 36B | 최신. Psyche **분산 학습망**으로 훈련 |

공통 강점 (에이전트 적합 이유):
- **시스템 프롬프트 준수** — 긴 워크로드에서 역할·규칙 드리프트 적음
- **구조화 출력(JSON)** — 형식 준수율 높음
- **Function Calling** — 포맷 내재화
- **낮은 거부율(steerability)** — 자율 실행 중 중단 적음

> 모델 일반 특성·선택 가이드는 [[Open-Source-Models]]. 이 노트는 **에이전트 활용**에 집중.

---

## 2. Function Calling 포맷 (핵심)

Hermes는 **XML 태그 기반 도구 호출 표준**으로 학습됐다. 외부 파싱 핵 없이 모델이 직접 `<tool_call>`을 안정적으로 emit한다.

```
① 시스템 프롬프트: <tools> 태그에 함수 정의(JSON Schema) 주입
② 모델 판단: 도구 필요 시 <tool_call> 태그로 {name, arguments} JSON 출력
③ 외부 실행: 런타임이 해당 함수 실제 실행
④ 결과 주입: <tool_response> 태그로 결과를 메시지 히스토리에 추가
⑤ 반복: 모델이 결과 평가 → 추가 호출 or 최종 답변
```

```text
# 시스템 프롬프트 (ChatML)
You are a function calling AI model. You are provided with function
signatures within <tools></tools> XML tags. You may call one or more
functions to assist with the user query...

<tools>
{"type":"function","function":{"name":"get_weather",
 "parameters":{"type":"object",
   "properties":{"city":{"type":"string"}},"required":["city"]}}}
</tools>

# 모델 출력 (도구 호출)
<tool_call>
{"name": "get_weather", "arguments": {"city": "Seoul"}}
</tool_call>

# 결과 주입 (다음 turn에 tool 역할로)
<tool_response>
{"temp": 24, "condition": "맑음"}
</tool_response>

# 모델 최종 답변
서울은 현재 24도, 맑습니다.
```

- 도구 호출 스키마: Pydantic `FunctionCall`(`name`, `arguments` 필수).
- 정의 위치 `<tools>`, 호출 `<tool_call>`, 결과 `<tool_response>`.
- vLLM([[vLLM]])·Ollama가 Hermes 채팅 템플릿을 내장 지원.

---

## 3. 구현 핵심 (3단계)

```python
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained("NousResearch/Hermes-3-Llama-3.1-8B")

tools = [{"type":"function","function":{
    "name":"get_weather",
    "parameters":{"type":"object",
        "properties":{"city":{"type":"string"}},"required":["city"]}}}]

messages = [
    {"role":"system","content":"You are a function calling AI model..."},
    {"role":"user","content":"서울 날씨 알려줘"},
]
# 1) 도구 목록을 chat template로 주입
prompt = tok.apply_chat_template(messages, tools=tools, add_generation_prompt=True)
# 2) 모델 생성 → <tool_call> 파싱
# 3) 도구 실행 결과를 messages에 role="tool"로 추가 후 재호출 (루프)
```

```
규칙:
  - 시스템 프롬프트에 번호 매긴 규칙으로 행동 제약
  - 모든 tool 실행 결과를 message 히스토리에 다시 넣고 재호출
  - <tool_call> JSON 파싱 실패 대비 (재시도/검증)
```

---

## 4. 에이전트 루프

```
사용자 질문
  → Hermes: <tool_call> 생성 (도구 필요 판단)
  → 런타임: 도구 실행 → <tool_response> 주입
  → Hermes: 결과 평가 → 추가 <tool_call> or 최종 답변
  → (반복, 종료 조건까지)
```

이 구조는 [[Agent-Harness]]의 에이전트 루프와 동일하고, [[MCP-Skill]]의 도구 연결과 결합 가능하다. Hermes의 차별점은 **포맷을 파인튜닝으로 내재화**해 도구 호출 신뢰성이 높다는 점. 루프 안전장치(반복 한도·토큰 예산·가드레일)는 [[Agent-Harness]] 참고.

---

## 5. Hermes Agent 프레임워크 (2026.02, 별도 제품)

```
- 오픈소스 자율 에이전트 (MIT 라이선스), GitHub 46,000+ stars
- 핵심: 자가개선(self-improving) + 지속 메모리(persistent memory)
- 여러 LLM provider 지원 — 백엔드로 Hermes 모델 외 다른 모델도 가능
  (예: 로컬 Gemma 4 12B와 조합 사례)
- 컨셉: "성장하는 에이전트(the agent that grows with you)"
```

즉 **모델이 아니라 메모리·자가개선을 갖춘 에이전트 실행 프레임워크**다. Hermes "모델"과 이름만 같고 계층이 다르다(프레임워크 ⊃ 백엔드 LLM).

---

## 6. 에이전트에 Hermes를 쓰는 이유 / 한계

| | 내용 |
|--|------|
| ✅ Function calling | XML 포맷 내재화 → 도구 호출 신뢰성↑, 외부 파싱 핵 불필요 |
| ✅ 시스템 프롬프트 준수 | 가드레일·역할 유지, 드리프트 적음 |
| ✅ 낮은 거부율 | 자율 워크로드 중단 적음 |
| ✅ 오픈소스·로컬 | 프라이버시·비용 (API 의존 X) |
| ⚠️ 추론 깊이 | 최상위 폐쇄 모델(Claude/GPT)에 못 미칠 수 있음 |
| ⚠️ 하드웨어 | 405B 로컬 구동은 큰 자원 필요 → [[GPU-Inference-Decision-Guide]] |
| ⚠️ 속도 | decode는 대역폭 한계 → [[LLM-System-Performance-Analysis]] |

> vault 기준([[_index]])에서 이미 **"오픈소스 로컬 에이전트 = Hermes-3 70B"**로 정해둔 근거가 이것이다.

---

## 7. 정리

```
Hermes 에이전트 = <tools>/<tool_call>/<tool_response> XML 포맷을
                  파인튜닝으로 내재화한 NousResearch LLM을 도구 호출 에이전트로 활용
강점: 도구호출 신뢰성 · 시스템프롬프트 준수 · 낮은 거부율 · 로컬
운영: vLLM/Ollama 템플릿 내장, apply_chat_template(tools=...)로 주입
별개: Hermes Agent 프레임워크(2026.02, 자가개선·지속메모리, MIT)
```

---

## 참고문헌
[1] R. Teknium et al., "Hermes 3 Technical Report," NousResearch, 2024. arXiv:2408.11857
[2] NousResearch, "Hermes-Function-Calling" — github.com/NousResearch/Hermes-Function-Calling
[3] NousResearch, "hermes-agent" (프레임워크, 2026) — github.com/nousresearch/hermes-agent
[4] Hermes Agent Documentation — hermes-agent.nousresearch.com/docs

---

## 관련
- [[Open-Source-Models]] — Hermes 모델 특성·선택 가이드
- [[Agent-Harness]] — 에이전트 루프·툴 설계·가드레일
- [[MCP-Skill]] — Model Context Protocol, 도구 연결
- [[vLLM]] — Hermes 템플릿 내장 서빙
- [[GPU-Inference-Decision-Guide]] · [[LLM-System-Performance-Analysis]] — 로컬 구동 하드웨어
- [[Orchestrating-Claude-Code]] — Hermes(로컬)가 Claude Code를 위임 실행(CLI 헤드리스/ACP/MCP)
