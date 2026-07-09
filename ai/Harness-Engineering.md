---
tags:
  - ai
  - agent
  - harness
  - harness-engineering
  - agentic-loop
created: 2026-06-24
---

# 하네스 엔지니어링 (Harness Engineering)

> [!summary] 한 줄 요약
> **Agent = Model + Harness.** 하네스는 LLM이 에이전트로 행동할 때 그를 둘러싼 *프롬프트·도구·규칙·샌드박스·검증기·피드백 루프*의 총체다. "하네스 엔지니어링"은 이 **모델 바깥 레이어를 모델만큼(때로는 더) 중요하게 다루는 정식 분야**로 2026년 부상했다. 핵심 주장: **에이전트 문제는 대개 모델 문제가 아니라 시스템(환경) 설계 문제다.** 모델은 생성하고, 둘러싼 레이어가 통제한다.

> 구현 메커니즘(루프 코드·툴 설계·컨텍스트 관리·가드레일)은 [[Agent-Harness]]. 이 노트는 그 위의 **개념·담론·왜 중요한가**.

---

## 1. 하네스란 무엇인가
모델은 텍스트(다음에 무엇을 할지)만 내놓는다. **행동은 하네스가 한다.** 하네스에 들어가는 것:

| 레이어 | 역할 |
|---|---|
| **에이전트 루프** | 호출→도구실행→결과 반영을 반복하는 엔진(§2) |
| **도구(tools)** | 모델의 의도를 실제 행동으로(웹·파일·셸·API) |
| **컨텍스트 관리** | 압축·편집·메모리·RAG로 긴 작업을 지탱 |
| **규칙·프롬프트** | 시스템 지시·역할·제약 |
| **샌드박스·권한** | 격리·승인 게이트 |
| **검증기·피드백 루프** | 테스트·린트·결과 검증으로 모델에 "틀렸다"를 알려줌 |
| **환경(map)** | 코드/자료 구조, 빠른 테스트, 팀 컨벤션 — 모델이 이해할 수 있는 작업장 |

---

## 2. 핵심 엔진 = 에이전트 루프
```
요청 → LLM 호출 → (tool_use?) → 하네스가 도구 실행 → tool_result 재투입 → 반복 → end_turn
```
- 모델은 stateless → 매 반복마다 전체 messages 재전송. 루프·툴 매칭·종료조건·상한이 하네스의 책임.
- 루프 자체의 구현·안전장치(반복 한도·토큰 예산·발산 감지)는 [[Agent-Harness]] §1·§7, Hermes 구현은 [[Hermes-Runtime-Internals]].
- 검색에 적용한 루프(Self-RAG·CRAG·반복검색)는 [[Agentic-RAG]].

---

## 3. 왜 하네스가 모델보다 중요한가
> [!important] 같은 모델, 다른 하네스 → 46% vs 80%
> 동일 Claude 모델을 동일 코딩 벤치마크에 돌려도 **에이전트 구조(하네스)에 따라 46% ↔ 80%**, 즉 **34%p 차이**가 났다. 점수를 가른 건 모델이 아니라 *모델을 감싼 래퍼*다.

- **"모델은 엔진, 승부는 그 바깥에서 난다."** 그리고 그 바깥은 **모델 벤더가 아니라 내가 만드는 것**이다.
- 팀이 "에이전트가 환각한다 / 코드를 이해 못 한다"고 할 때, 원인은 거의 모델이 아니다 — 에이전트가 **지도도, 빠른 테스트도, 버그 재현 수단도, 팀 컨벤션도 없고, 틀렸다고 알려주는 피드백도 없는** 환경에서 일하기 때문이다. 그 환경이면 새로 온 사람도 똑같이 헤맨다.
- 결론: 대부분의 LLM 실패는 **모델 문제가 아니라 시스템 설계 문제**. 모델 업그레이드를 좇기보다 하네스(환경·피드백·도구)에 투자하는 게 레버리지가 크다.

---

## 4. 좋은 하네스의 조건 (실무 체크)
- **빠른 피드백 루프**: 테스트·린트·타입체크를 에이전트가 즉시 돌려 스스로 교정.
- **환경 정비**: 코드맵·인덱스·검색(RAG)·컨벤션 문서를 제공해 모델이 "이해 가능한" 작업장을 만든다.
- **도구 설계**: 넓힐 땐 bash, 통제할 땐 전용 툴로 승격(승인 게이트·감사). [[Agent-Harness]] §3.
- **컨텍스트 위생**: 압축·편집·메모리·RAG로 토큰 폭주와 노이즈를 막는다.
- **가드레일**: 루프 상한·토큰 예산·발산 감지·샌드박스·프롬프트 인젝션 방어.
- **관측성**: 모든 툴 호출·토큰·지연을 로깅해 실패를 분석.

---

## 5. 이 볼트의 맥락
- **런타임 = 하네스**: [[Autonomous-Agent-Runtimes]]의 "모델(두뇌) vs 런타임(몸체)" 구분이 곧 모델 vs 하네스. Hermes·OpenClaw는 하네스를 제공하는 런타임이고, 그 자기관리(압축·가드레일·샌드박스)가 [[Hermes-Runtime-Internals]].
- **이 볼트의 RAG도 하네스 사례**: 태그가중 검색→주입→1회 생성이라는 단순 하네스가 에이전트 루프보다 속도·신뢰성에서 유리([[Obsidian-RAG-System]]·[[RAG-Latency-Reality]]). 라우팅 엔진을 `claude -p`로 감싼 것도 하네스 설계([[RAG-Routing-Engine-Comparison]]).
- **자율 루프 운영**: cron·`/loop`로 도는 무인 루프도 하네스(가드레일·재개·배포)가 품질을 가른다.

---

## 6. 관련
- [[Agent-Harness]] — 하네스 **구현**: 루프 코드·툴 설계·컨텍스트 관리·가드레일·멀티에이전트
- [[Hermes-Runtime-Internals]] — 런타임 하네스 자기관리(압축·tool-loop 가드레일·샌드박스)
- [[Autonomous-Agent-Runtimes]] — 모델 vs 런타임(=하네스) 계층
- [[Agentic-RAG]] — 검색에 적용한 에이전트 루프
- [[Agent-Security]] — 하네스 보안(권한·샌드박싱·인젝션 방어)
- [[Prompt-Engineering]] · [[Context-Assembly-Compression]] — 하네스의 프롬프트·컨텍스트 레이어

---

## 출처
- [Why the harness matters more than the model — Harness Engineering](https://jmlopezdona.github.io/ai-coding-agents-harness/01-why-harness/)
- [Harness Engineering: Why the System Around the LLM Matters More Than the Model (Medium, 2026.04)](https://medium.com/@advait.darbare9/harness-engineering-why-the-system-around-the-llm-matters-more-than-the-model-bf7ae71d370f)
- [Agent = Model + Harness (Cobus Greyling, 2026.06)](https://cobusgreyling.medium.com/agent-model-harness-0d018f3d5014)
- [Harness Engineering Is Now a Formal Discipline — 6 Findings (MindStudio)](https://www.mindstudio.ai/blog/harness-engineering-formal-discipline-6-findings-ai-agents)
- [The Anatomy of an Agent Harness (LangChain)](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
