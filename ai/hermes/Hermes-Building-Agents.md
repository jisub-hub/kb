---
tags:
  - ai
  - agent
  - hermes
  - how-to
  - orchestration
created: 2026-06-24
---

# Hermes 에이전트 — 용도와 만드는 법

> [!summary] 한 줄 요약
> 이 노트는 두 부분이다. **A.** Hermes 에이전트가 단순 LLM 호출과 무엇이 다르고 언제·왜 만드나(툴·메모리·스킬·루프), 그리고 **안 만들어도 되는 경우**. **B.** 실제로 하나를 만드는 단계 — 정체성(시스템 프롬프트) → 모델 백엔드 → 툴셋 → 스킬 → 플러그인 → 메모리 → 배포 표면 → 멀티 프로필. 근거는 vault의 Hermes 기록([[Hermes-Agent]] [[Hermes-Skills]] [[Hermes-Plugins]] [[Hermes-Deployment-Surface]] [[Hermes-Runtime-Internals]] [[Local-LLM-API-Deployment]] [[Tool-Search]]) 이며, 노트에 없는 사실은 *(일반 에이전트 원리)* 로 표시한다.

---

## A. Agent의 용도 — 언제·왜 만드나

### A-1. 단순 LLM 호출과 무엇이 다른가

순수 LLM 호출은 "프롬프트 in → 텍스트 out" 1회다. 에이전트는 여기에 **네 가지 레이어**를 더해, 외부 세계와 상호작용하며 스스로 반복한다.

| 레이어 | 무엇 | 근거 노트 |
|--------|------|-----------|
| **툴(Function Calling)** | `<tools>`/`<tool_call>`/`<tool_response>` XML로 실제 함수를 호출·결과 회수 | [[Hermes-Agent]] |
| **루프** | 도구 결과를 평가해 추가 호출 또는 최종 답변까지 반복 | [[Hermes-Agent]] §4 |
| **메모리** | 선언적(사실)·절차적(스킬)·세션 기록의 3계층 기억으로 세션을 넘어 학습 | [[Hermes-Skills]] §6, [[Agent-Persistent-Memory]] |
| **스킬** | on-demand로 펼치는 절차 문서(progressive disclosure) | [[Hermes-Skills]] |

핵심 차이를 한 줄로:

```
LLM 호출   : 입력 → (한 번의) 텍스트 출력
에이전트   : 입력 → [도구 호출 → 결과 주입 → 재평가]를 반복 → 도구·메모리·스킬을 동원해 작업을 "수행"
```

> Hermes의 차별점은 도구 호출 포맷을 **파인튜닝으로 내재화**해 외부 파싱 핵 없이 `<tool_call>`을 안정적으로 emit한다는 것 — 도구 호출 신뢰성·시스템 프롬프트 준수·낮은 거부율이 강점이다([[Hermes-Agent]] §2·§6).

### A-2. 어떤 작업에 적합한가

읽은 노트가 다루는 실제 용도들:

- **자동화 / 상시 어시스턴트** — 메신저 봇(Telegram·Discord·Slack·WhatsApp)으로 상주시켜 채팅으로 호출. cron 예약 실행과 결합 ([[Hermes-Deployment-Surface]] §2·§6).
- **코딩 위임** — 큰 하위작업을 격리·병렬 처리. `delegate_task`로 자식 에이전트에 위임하거나, 외부 코딩 에이전트(Claude Code)를 오케스트레이션 ([[Hermes-Runtime-Internals]] §1, [[Orchestrating-Claude-Code]]).
- **프라이버시·비용 민감한 로컬 개발** — 로컬 모델로 비용 0·오프라인·데이터 로컬 ([[Local-LLM-API-Deployment]] §6).
- **RAG·웹 등 외부 도구 결합** — MCP로 남이 만든 도구를, 플러그인으로 내 도구를 붙임 ([[Hermes-Plugins]] §4, [[MCP-Skill]]).
- **프론트엔드 백엔드** — OpenAI 호환 API 서버로 노출해 Open WebUI·LobeChat 등 수백 개 UI의 *풀 툴셋 백엔드*로 사용 ([[Hermes-Deployment-Surface]] §1).

### A-3. 안 만들어도 되는 경우

에이전트는 공짜가 아니다 — 위임 비용(자식 부팅·컨텍스트 재구성), 툴 스키마의 컨텍스트 점유, 루프 폭주·표류 위험을 동반한다.

```
에이전트가 과한 경우:
  - 도구·외부 상태가 필요 없는 1회성 변환(요약·번역·분류) → 그냥 LLM 호출
  - 잔작업까지 서브에이전트로 쪼개기 → 위임 비용이 이득을 넘음 ([[Hermes-Runtime-Internals]] §1)
  - 툴 몇 개뿐인데 Tool Search부터 켜기 → 고정비용(브리지 ~300토큰+왕복) > 절약분 ([[Tool-Search]] §6)
  - 다수 동시 사용자 프로덕션을 Mac 로컬로 → 동시성·HA 부족 ([[Local-LLM-API-Deployment]] §6)
```

> 판단 기준: **외부 세계와의 상호작용·반복·기억 중 하나라도 진짜 필요한가?** 아니면 단순 LLM 호출로 충분하다.

---

## B. Agent 만드는 법 — 단계별

전제: Hermes는 `~/.hermes` 기준으로 동작하며, 확장(스킬·플러그인·MCP)은 모두 이 디렉터리 아래에 둔다.

```
정체성 → 모델 백엔드 → 툴셋 → 스킬 → 플러그인 → 메모리 → 배포 표면 → 멀티 프로필
(누구) (무엇으로)  (무슨도구) (절차)  (내도구) (기억)   (어디로 노출)   (여럿 운영)
```

### B-1. 정체성 정의 — 시스템 프롬프트·역할

도구 호출 에이전트의 시스템 프롬프트는 **함수 시그니처를 `<tools>` 태그에 주입**하고, 번호 매긴 규칙으로 행동을 제약한다([[Hermes-Agent]] §2·§3).

```text
# 시스템 프롬프트 (ChatML)
You are a function calling AI model. You are provided with function
signatures within <tools></tools> XML tags...

<tools>
{"type":"function","function":{"name":"get_weather",
 "parameters":{"type":"object",
   "properties":{"city":{"type":"string"}},"required":["city"]}}}
</tools>
```

```
규칙(노트 기준):
  - 시스템 프롬프트에 번호 매긴 규칙으로 행동 제약
  - 모든 tool 실행 결과를 message 히스토리에 다시 넣고 재호출
  - <tool_call> JSON 파싱 실패 대비(재시도/검증)
```

> Hermes 모델은 시스템 프롬프트 준수율이 높아 긴 워크로드에서 역할·규칙 드리프트가 적다([[Hermes-Agent]] §1) — 정체성을 시스템 프롬프트에 새기는 것이 효과적인 이유.

### B-2. 모델 백엔드 선택 — 로컬/원격

> 상세: [[Local-LLM-API-Deployment]]

**Hermes 모델이 안전한 기본값이지만 강제는 아니다.** OpenAI 호환 API로 추상화하면 모델 교체가 `base-url` 한 줄이다.

| 목적 | 권장 모델 | 비고 |
|------|----------|------|
| 도구 호출 신뢰성 최우선 | Hermes 3/4 | 포맷 내재화·낮은 거부율 |
| 코딩/수학 강한 에이전트 | Qwen2.5-Coder | tool calling 지원 |
| 범용·최신 | Llama 3.3 70B | 생태계 넓음 |
| 소형·빠름(개발) | Hermes 3 8B / Qwen2.5 7B | 8B도 function calling 충분 |

Mac 로컬 서빙 스택: **Ollama**(가장 쉬움, 템플릿+tool calling+OpenAI API 내장), `mlx_lm.server`(MLX), `llama.cpp`(GGUF). vLLM은 CUDA 전용이라 Mac 불가.

```bash
# Ollama — 가장 빠른 시작
ollama pull hermes3:8b
ollama serve                       # OpenAI 호환 API :11434
# → POST http://localhost:11434/v1/chat/completions (tools 지원)
```

```python
# OpenAI SDK 그대로, base_url만 로컬로
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

> 설계 원칙: **처음부터 OpenAI 호환으로 추상화** → "Mac 개발 → 서버(vLLM/NVIDIA) 이전"이 `base-url` 교체로 끝난다([[Local-LLM-API-Deployment]] §7).

### B-3. 툴셋 장착

에이전트의 손발. Hermes에는 항상 직접 노출되는 **코어 툴셋**이 있다([[Tool-Search]] §4):

```
_HERMES_CORE_TOOLS (항상 직접 노출):
  terminal · read_file · write_file · patch · search_files · todo ·
  memory · web_search · execute_code · delegate_task · session_search · clarify ...
```

플랫폼별 전용 툴셋도 있다 — 게이트웨이 배포 시 `platform_toolsets`로 부여(discord→hermes-discord 등, [[Hermes-Deployment-Surface]] §2).

도구가 많아지면(MCP 서버 여럿·툴 수십 개) 스키마가 매 턴 컨텍스트를 잠식한다 → **Tool Search**(progressive disclosure)로 완화. 기본 `auto`라 임계(10%) 미만에선 자동으로 pass-through되니 소수 툴 세션엔 켜지 마라([[Tool-Search]] §3·§6).

```yaml
tools:
  tool_search:
    enabled: auto          # auto(기본) | on(항상) | off
    threshold_pct: 10      # auto 활성 임계(%)
    search_default_limit: 5
    max_search_limit: 20
```

### B-4. 스킬 추가 — 재사용 절차 지식

> 상세: [[Hermes-Skills]]

스킬은 **실행 코드가 아니라 on-demand로 펼쳐 읽는 절차 문서**다. 평소엔 이름+설명만 보이고(Level 0), 모델이 필요하다 판단하면 `skill_view`로 본문을 로드해 일반 툴로 수행한다. 모든 스킬은 `~/.hermes/skills/`에 산다.

```markdown
---
name: my-skill
description: Brief description     # ← Level 0 목록 노출, 모델 선택 신호(가장 중요)
platforms: [macos, linux]
metadata:
  hermes:
    requires_toolsets: [terminal]   # 조건부 활성
    config: [{key: my.setting}]     # config.yaml 연동(로드 시 주입)
---
# Skill Title
## When to Use       # ← 두 번째 선택 신호
## Procedure         # ← 로드되면 이 절차가 모델 컨텍스트로
```

핵심: **`description`과 `When to Use`를 명확히** 써야 모델이 올바르게 로드한다. 에이전트는 `skill_manage`로 자기 워크플로를 스킬로 저장(절차 기억)할 수도 있다([[Hermes-Skills]] §6).

### B-5. 플러그인으로 커스텀 툴·훅

> 상세: [[Hermes-Plugins]]

"나/팀/한 프로젝트용 커스텀 툴"이 필요하면 플러그인이 보통 정답이다. 코어 코드를 고치지 않고 `~/.hermes/plugins/<name>/`에 디렉터리를 떨궈 두면 시작 시 자동 로드되어 내장 툴과 나란히 모델에 노출된다.

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml      # 매니페스트(name·version·description)
├── __init__.py      # register(ctx) — 스키마↔핸들러 배선 진입점
├── schemas.py       # 툴 스키마(LLM이 보는 것)
└── tools.py         # 툴 핸들러(실제 실행 코드)
```

```python
# __init__.py
def register(ctx):
    ctx.register_tool(schema, handler)              # 툴 등록(스키마↔핸들러)
    ctx.register_hook("pre_tool_call", my_observer) # 훅 등록(생명주기 콜백)
```

확장 3종 구분 — **코드로 새 동작 = Plugin / 매뉴얼 = Skill / 외부 도구 연결 = MCP**. 플러그인은 인프로세스 실행이라 빠르지만 격리가 없으니 신뢰하는 코드만; 격리가 필요하면 shell hook을 쓴다([[Hermes-Plugins]] §4·§5).

### B-6. 메모리 설정

> 상세: [[Agent-Persistent-Memory]]

세 기억이 합쳐져 "세션을 넘는 학습"을 이룬다([[Hermes-Skills]] §6):

```
선언적 기억(MEMORY.md/USER.md) : 무엇을 안다(환경·선호·사실)    — 항상 주입, 작음
절차 기억(skills)              : 어떻게 한다(재사용 워크플로)    — on-demand 로드, 큼
세션 기록(session_search)      : 무슨 일이 있었나(과거 대화)     — 검색
```

장시간 세션은 컨텍스트 압축으로 중간을 요약하는데, 이때 **정보 손실**이 있으니 영구적으로 필요한 사실은 메모리/파일에 적어 둬야 한다([[Hermes-Runtime-Internals]] §2). 게이트웨이 배포 시 세션은 per-user/platform으로 갈리고 각자 영속 메모리를 가진다([[Hermes-Deployment-Surface]] §3).

### B-7. 배포 표면 선택 — CLI → API 서버 → 게이트웨이

> 상세: [[Hermes-Deployment-Surface]]

같은 에이전트(툴·메모리·스킬)를 **어느 표면으로 내보내느냐**의 차이다.

```
개발   : CLI / TUI (로컬 대화·작업)
위임   : 오케스트레이터 → Claude Code 등 ([[Orchestrating-Claude-Code]])
자율   : cron 예약 · Kanban
배포   : ① API 서버(프론트엔드 백엔드)  ② 게이트웨이(메신저 봇)
```

**① OpenAI 호환 API 서버** — 에이전트 전체(툴·메모리·스킬)를 HTTP로 노출. 스트리밍 시 tool progress가 inline.

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer <key>" -d '{...}'
# CORS·키 설정(API_SERVER_*), /v1/chat/completions 등 엔드포인트
```

> [!important] [[Local-LLM-API-Deployment]]와 구분
> Local-LLM-API는 **raw 모델**(mlx_lm.server 등)을 노출. 여기 API 서버는 **에이전트 전체**를 노출한다 — 백엔드가 "모델"이 아니라 "에이전트".

**② 멀티플랫폼 게이트웨이** — Telegram/Discord/Slack/WhatsApp 봇으로 상주. `platform_toolsets`로 플랫폼별 전용 툴셋을 부여하고, gateway hooks가 이 표면에서 발화(로깅·알림·웹훅).

세션 모델:

```
group_sessions_per_user: true   — 사용자별 세션 분리(대화·메모리 격리)
max_concurrent_sessions: null   — 동시 세션 상한(기본 무제한)
session_reset: idle/at_hour     — 리셋 기준
```

### B-8. 멀티 프로필 운영

> 상세: [[Hermes-Deployment-Surface]] §4

```
profile = 독립된 봇 토큰·세션·메모리·정체성 묶음
한 머신에서 여러 프로필을 managed service(launchd/systemd)로 동시 가동
  → "여러 named 봇"(inbox-triage · ops-review ...) 각자 상주
운영 관심사: 일괄 시작 · 프로필 통합 로그 · sleep 방지 · 서비스 복구
```

### B-9. 장시간 자율 운영 — 자기관리 4종

> 상세: [[Hermes-Runtime-Internals]]

에이전트를 오래 자율로 돌리려면 스스로를 관리하는 4가지 런타임 장치가 받친다. 기본값은 노트 기준:

| 장치 | 역할 | 핵심 기본값 |
|------|------|------------|
| `delegate_task` 서브에이전트 | 작업 격리·병렬, 부모 컨텍스트 보호 | 동시 최대 3, `delegation.max_iterations: 50` |
| 컨텍스트 압축 | 토큰 한계 전 중간 요약 | `threshold: 0.5`, `target_ratio: 0.2`, 앞3·뒤20 보호 |
| tool-loop 가드레일 | 같은 실패·무진전 반복 차단 | 경고 on, `hard_stop_enabled: false`(보수적) |
| 터미널 샌드박스 | 자원·수명·타임아웃으로 실행 격리 | `lifetime_seconds: 300`, `timeout: 180` |

> [!danger] 서브에이전트는 부모 대화를 전혀 모른다
> 자식은 완전히 새 대화로 시작 — 파일 경로·에러 전문·환경을 `goal`/`context`에 **전부 명시**해야 한다([[Hermes-Runtime-Internals]] §1).

```python
delegate_task(tasks=[
    {"goal": "Research topic A", "toolsets": ["web"]},
    {"goal": "Fix the build",   "toolsets": ["terminal", "file"]},
])
```

> 무인 자동 환경이라면 `hard_stop_enabled`를 켜는 것을 고려([[Hermes-Runtime-Internals]] §3).

---

## C. 정리

```
용도 판단 : 외부 상호작용·반복·기억이 필요하면 에이전트, 아니면 단순 LLM 호출
만드는 법 : 시스템 프롬프트(정체성) → OpenAI 호환 모델 백엔드 → 코어 툴셋(+Tool Search)
            → 스킬(절차) → 플러그인(내 도구) → 3계층 메모리
            → 배포(CLI/API/게이트웨이) → 멀티 프로필
안정성    : delegate_task · 압축 · 가드레일 · 샌드박스 4종이 장시간 자율을 받친다
```

---

## 관련
- [[Hermes-Agent]] — function calling·툴 루프·프레임워크(토대)
- [[Hermes-Skills]] — 스킬 로딩·progressive disclosure·절차 기억
- [[Hermes-Plugins]] — 플러그인으로 커스텀 툴·훅 추가
- [[Hermes-Deployment-Surface]] — API 서버·게이트웨이·멀티 프로필
- [[Hermes-Runtime-Internals]] — delegate_task·압축·가드레일·샌드박스
- [[Local-LLM-API-Deployment]] — 모델 백엔드(로컬/원격, OpenAI 호환 추상화)
- [[Tool-Search]] — 툴 과부하 시 점진적 공개
- [[Agent-Persistent-Memory]] — 선언적 기억(사실) 계층
- [[Orchestrating-Claude-Code]] — 외부 코딩 에이전트 위임
