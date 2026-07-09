---
tags:
  - ai
  - agent
  - hermes
  - overview
  - orchestration
created: 2026-06-24
---

# Hermes 종합 개요 — 툴 장착 자율 에이전트 런타임

> [!summary] 한 줄 요약
> **Hermes**(`~/.hermes` 기준)는 LLM에 터미널·파일·웹·메모리·스킬 같은 **풀 툴셋**을 장착해 도구 호출 루프로 일을 시키는 **자율 에이전트 런타임**이다. 백엔드 LLM(지능)과 실행 루프(런타임)를 분리해, 같은 에이전트를 CLI 개발부터 cron 자율 실행, API·메신저 봇 배포까지 여러 표면으로 내보낸다. (NousResearch의 동명 **Hermes 모델**과는 계층이 다른 별개 제품 — [[Hermes-Agent]] §0 참조.)

---

## 1. 모델 vs 런타임 — 두 계층을 먼저 분리

에이전트를 이해할 때 가장 중요한 구분은 **"두뇌(모델)"와 "몸체(런타임)"**가 다른 계층이라는 것이다.

| 계층 | 무엇 | 역할 | 예 |
|------|------|------|------|
| **런타임(몸체)** | 실행 루프·도구·메모리·자가개선 | 도구 실행, 컨텍스트 관리, 액션 수행 | Hermes Agent · OpenClaw |
| **모델(두뇌)** | LLM | 추론·도구 호출 판단 | Hermes 모델 · Claude · GPT · Qwen · Gemma |

런타임은 백엔드 LLM을 **갈아끼운다**. 그래서 "Hermes 모델로 OpenClaw를 대체한다"는 식의 비교는 범주 오류이고, 올바른 관계는 **런타임 + (그 안에 꽂는) 모델**의 조합이다. 이 계층 분리가 [[Orchestrating-Claude-Code]]의 "로컬 모델을 플래너로, 무거운 코딩만 Claude Code에 위임" 패턴의 근거이기도 하다. 런타임 간 비교 맥락은 [[Autonomous-Agent-Runtimes]].

---

## 2. 생애주기 4단계

같은 에이전트(같은 툴·메모리·스킬)를 **어느 표면으로 내보내느냐**의 차이일 뿐, 단계가 올라갈수록 사람 개입이 줄어든다.

| 단계 | 표면 | 설명 | 관련 노트 |
|------|------|------|----------|
| ① 개발 | CLI / TUI | 로컬에서 직접 대화하며 작업·검증 | — |
| ② 위임 | 오케스트레이터 → 외부 에이전트 | 로컬 모델이 플래너가 되어 무거운 작업만 Claude Code 등에 헤드리스로 위임·검증 | [[Orchestrating-Claude-Code]] |
| ③ 자율 | cron · Kanban | 사람 없이 예약 실행되고, 여러 named 프로필이 보드로 협업 | [[Agent-Scheduled-Execution]] · [[Agent-Kanban]] |
| ④ 배포 | API 서버 · 게이트웨이 | 에이전트 전체를 OpenAI 호환 API나 메신저 봇으로 상시 노출 | [[Hermes-Deployment-Surface]] |

> [!note] "에이전트→에이전트 위임"과 "에이전트→사용자 노출"은 다른 축
> ② 위임([[Orchestrating-Claude-Code]])은 다른 에이전트에게 일을 넘기는 축이고, ④ 배포([[Hermes-Deployment-Surface]])는 사람·프론트엔드에 닿는 축이다. 혼동하지 말 것.

---

## 3. 핵심 서브시스템

### 3.1 에이전트 루프 · Function Calling — [[Hermes-Agent]]
모든 동작의 토대. 시스템 프롬프트에 도구 정의를 주입하면 모델이 도구 호출을 생성하고, 런타임이 실제 실행한 뒤 결과를 다시 히스토리에 넣어 재호출하는 **반복 루프**다. (NousResearch Hermes 모델은 `<tools>`/`<tool_call>`/`<tool_response>` XML 포맷을 파인튜닝으로 내재화해 이 도구 호출 신뢰성이 높다.)

### 3.2 툴셋
터미널·파일·웹·메모리·스킬 등 에이전트가 손에 쥔 능력의 묶음. 위임받은 서브에이전트나 cron 작업, 게이트웨이 플랫폼별로 **툴셋을 좁혀 줄 수 있다**(예: 플랫폼별 `platform_toolsets`).

### 3.3 스킬(절차 기억) — [[Hermes-Skills]]
함수처럼 "호출"되는 게 아니라 필요할 때 펼쳐 읽는 **on-demand 지식 문서**. 평소엔 이름·설명만 보이고(progressive disclosure), 모델이 본문을 로드해 그 절차대로 일반 툴로 수행한다. `skill_manage`로 에이전트가 자기 워크플로를 스킬로 저장 = **절차 기억**. 모든 스킬은 `~/.hermes/skills/`에 산다.

### 3.4 플러그인(코드 확장) — [[Hermes-Plugins]]
코어를 고치지 않고 `~/.hermes/plugins/<name>/`에 매니페스트 + Python을 떨궈 **커스텀 도구·훅**을 붙이는 1차 확장 경로. `register(ctx)`로 "모델이 보는 스키마"와 "실제 핸들러"를 배선한다. 인프로세스 실행이라 빠르지만 격리가 없어 신뢰 코드만. (확장 3종: 코드=Plugin / 매뉴얼=Skill / 외부도구=MCP)

### 3.5 영속 메모리 — [[Agent-Persistent-Memory]]
세션을 넘어 사실(환경·선호)을 기억하는 **선언적 기억**(스킬의 절차 기억과 대비). 배포 표면에서는 per-user/platform으로 분리·격리된다.

### 3.6 세션 모델 — [[Hermes-Deployment-Surface]] §3
사용자·플랫폼별로 세션이 갈리고(`group_sessions_per_user`), 각 세션이 자체 메모리를 가진다. 동시 세션 상한·idle/시각 기준 리셋 등을 설정.

### 3.7 훅(생명주기 이벤트) — [[Agent-Event-Hooks]]
도구 호출 전후·LLM 호출 전 등 생명주기 지점에서 발화해 동작을 **관측·차단·증강**한다. 플러그인이 도구와 함께 훅을 등록하는 일이 흔하고, 게이트웨이 표면에서도 gateway hooks가 로깅·알림·웹훅으로 발화한다.

### 3.8 서브에이전트 위임 — [[Hermes-Runtime-Internals]] · [[Orchestrating-Claude-Code]]
- **내부**: `delegate_task`가 격리된 컨텍스트·제한된 툴셋·자체 터미널을 가진 자식 에이전트를 띄워 병렬 실행하고 "최종 요약"만 회수한다. 단, 자식은 부모 대화를 전혀 모르므로 `goal`·`context`에 모든 걸 명시해야 한다.
- **외부**: 로컬 오케스트레이터가 `claude -p` 헤드리스로 Claude Code에 위임하고 결과를 검증·재위임한다.

### 3.9 게이트웨이(배포 표면) — [[Hermes-Deployment-Surface]]
① OpenAI 호환 **API 서버**(Open WebUI·LobeChat 등이 *에이전트 전체*를 백엔드로 사용 — raw 모델 노출인 [[Local-LLM-API-Deployment]]와 구분), ② Telegram/Discord/Slack/WhatsApp **메신저 봇 게이트웨이**.

### 3.10 예약 실행 — [[Agent-Scheduled-Execution]]
cron으로 사람 없이 작업을 돌리고, 작업에 스킬을 붙일 수 있으며(`--skill`), 결과를 게이트웨이 타깃으로 전달한다.

### 3.11 멀티프로필 협업 — [[Agent-Kanban]]
profile = 독립된 봇 토큰·세션·메모리·정체성 묶음. 한 머신에서 여러 named 봇(inbox-triage·ops-review 등)을 동시 가동하고 Kanban 보드로 협업시킨다.

---

## 4. 장시간 자율을 떠받치는 런타임 자기관리

[[Hermes-Runtime-Internals]]에 따르면 에이전트가 장시간 자율로 돌려면 **스스로를 관리하는 4종 장치**가 필요하다.

| 장치 | 역할 |
|------|------|
| 서브에이전트(`delegate_task`) | 작업을 격리·병렬화해 부모 컨텍스트를 지킴 |
| 컨텍스트 압축 | 토큰 한계 전 중간을 요약(앞뒤 보호, 손실 주의) |
| tool-loop 가드레일 | 같은 실패·무진전 반복을 카운터로 차단(경고 우선) |
| 터미널 샌드박스 | 자원·수명·타임아웃으로 실행을 가둠 |

하나라도 없으면 표류·폭주·OOM으로 이어진다. 일반 루프 이론은 [[Agent-Harness]].

---

## 5. 다른 런타임과의 위치

Hermes Agent는 OpenClaw와 **같은 카테고리(자율 에이전트 런타임)**다. 둘은 경쟁이 아니라 레이어가 겹치는 두 런타임이고, 모델(Hermes/Qwen/Claude)은 그 아래 갈아끼우는 부품이다. 메시징 비서·액션 생태계 성숙도는 OpenClaw가, NousResearch 모델 긴밀 통합은 Hermes Agent가 강점이다. 상세 비교·선택 가이드는 [[Autonomous-Agent-Runtimes]].

```
런타임(OpenClaw / Hermes Agent)  ─ LLM을 갈아끼움
        └─ 모델(Hermes / Claude / GPT / Qwen / Gemma)
```

---

## 6. 관련

- [[Hermes-Agent]] — function calling 포맷·에이전트 루프, 두 "Hermes"(모델 vs 프레임워크) 구분
- [[Hermes-Runtime-Internals]] — 서브에이전트·압축·가드레일·샌드박스(자기관리 4종)
- [[Hermes-Skills]] — 절차 기억(스킬) 로딩·progressive disclosure
- [[Hermes-Plugins]] — 코드로 도구·훅 추가(확장의 근본 메커니즘)
- [[Hermes-Deployment-Surface]] — API 서버·게이트웨이(에이전트→사용자 노출)
- [[Orchestrating-Claude-Code]] — 오케스트레이터 → Claude Code 위임
- [[Agent-Persistent-Memory]] — 선언적 기억(사실)
- [[Agent-Event-Hooks]] — 생명주기 훅(관측·차단·증강)
- [[Agent-Scheduled-Execution]] — cron 예약 실행
- [[Agent-Kanban]] — 멀티프로필 봇 협업
- [[Autonomous-Agent-Runtimes]] — 모델 vs 런타임, OpenClaw 비교
- [[Local-LLM-API-Deployment]] — raw 모델 OpenAI API(에이전트 전체 노출과 구분)
- [[Agent-Harness]] — 에이전트 루프·가드레일 일반 이론
