---
tags:
  - ai
  - agent
  - hermes
  - configuration
  - reference
created: 2026-06-24
---

# Hermes 설정 레퍼런스 (`~/.hermes`)

> [!summary] 한 줄 요약
> 여러 노트에 흩어진 Hermes(`~/.hermes`) 설정·환경변수·런타임 파라미터를 **주제별 표**로 통합한 단일 레퍼런스. 각 항목의 **출처 노트**를 위키링크로 표기해 추적 가능하게 했다. **근거 기반만** 수록 — 읽은 노트에 실제 등장한 값만 넣고, 노트에 없는 기본값은 "(미명시)"로 둔다.

> 개념 설명은 출처 노트에 있다. 이 노트는 **설정값 인덱스**.

---

## 1. `config.yaml` — 런타임 핵심 키

### 1.1 터미널 샌드박스 (`terminal:`)

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `terminal.backend` | `local` | 실행 백엔드 (`local` 또는 `container`) | [[Hermes-Runtime-Internals]] |
| `terminal.container_cpu` | `1` | 컨테이너 CPU 수 | [[Hermes-Runtime-Internals]] |
| `terminal.container_memory` | `5120` (MB) | 컨테이너 메모리 상한 | [[Hermes-Runtime-Internals]] |
| `terminal.container_disk` | `51200` (MB) | 컨테이너 디스크 상한 | [[Hermes-Runtime-Internals]] |
| `terminal.container_persistent` | `true` | 컨테이너 지속 여부 | [[Hermes-Runtime-Internals]] |
| `terminal.lifetime_seconds` | `300` | 터미널 세션 수명(초) | [[Hermes-Runtime-Internals]] |
| `terminal.timeout` | `180` | 명령 1회 타임아웃(초) | [[Hermes-Runtime-Internals]] · [[Orchestrating-Claude-Code]] |
| `terminal.home_mode` | `auto` | 홈 디렉터리 모드 | [[Hermes-Runtime-Internals]] |

### 1.2 코드 실행 (`code_execution:`)

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `code_execution.max_tool_calls` | `50` | 코드 실행 시 최대 툴 호출 수 | [[Hermes-Runtime-Internals]] |
| `code_execution.timeout` | `300` | 코드 실행 타임아웃(초) | [[Hermes-Runtime-Internals]] · [[Orchestrating-Claude-Code]] |

> [!warning] 위임 타임아웃 함정
> `terminal.timeout`(180)·`code_execution.timeout`(300)은 `claude -p` 등 **외부 위임**에도 그대로 적용 — 긴 작업이 중간에 잘린다. 대응: 작업 쪼개기 / 한도 상향 / 백그라운드+폴링 / 스트리밍. → [[Orchestrating-Claude-Code]] §3.2

### 1.3 컨텍스트 압축 (`compression:`)

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `compression.enabled` | `true` | 압축 활성화 | [[Hermes-Runtime-Internals]] |
| `compression.threshold` | `0.5` | 컨텍스트의 50% 차면 압축 발동 | [[Hermes-Runtime-Internals]] |
| `compression.target_ratio` | `0.2` | 압축 후 목표 = 임계 토큰의 20% | [[Hermes-Runtime-Internals]] |
| `compression.protect_first_n` | `3` | 앞 N개 메시지 보존(정체성·초기 지시) | [[Hermes-Runtime-Internals]] |
| `compression.protect_last_n` | `20` | 최근 N개 메시지 보존(진행 맥락) | [[Hermes-Runtime-Internals]] |

### 1.4 tool-loop 가드레일 (`tool_loop_guardrails:`)

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `tool_loop_guardrails.warnings_enabled` | `true` | 경고 활성화 | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.hard_stop_enabled` | `false` | 강제 중단(보수적으로 기본 끔) | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.warn_after.exact_failure` | `2` | 동일 호출·동일 에러 반복 경고 임계 | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.warn_after.same_tool_failure` | `3` | 한 툴 반복 실패 경고 임계 | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.warn_after.idempotent_no_progress` | `2` | 무진전 반복 경고 임계 | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.hard_stop_after.exact_failure` | `5` | 동일 실패 하드스톱 임계 | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.hard_stop_after.same_tool_failure` | `8` | 한 툴 실패 하드스톱 임계 | [[Hermes-Runtime-Internals]] |
| `tool_loop_guardrails.hard_stop_after.idempotent_no_progress` | `5` | 무진전 하드스톱 임계 | [[Hermes-Runtime-Internals]] |

> 카운터 의미: **exact_failure**=같은 호출·같은 에러, **same_tool_failure**=한 툴이 인자 달라도 계속 실패, **idempotent_no_progress**=상태 변화 없는 헛돌기. → [[Hermes-Runtime-Internals]] §3

### 1.5 서브에이전트 위임 (`delegation:`)

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `delegation.max_iterations` | `50` | 자식 1개의 루프 한도 | [[Hermes-Runtime-Internals]] |
| 병렬 배치 동시 실행 | `3` (설정 가능, 하드 상한 없음) | `delegate_task` 동시 자식 수 | [[Hermes-Runtime-Internals]] |

> 키 이름은 노트에 명시되지 않음(값·동작만 서술). `delegate_task(tasks=[{goal, context, toolsets}, ...])` 형태. 자식은 부모 대화를 모르며 `goal`·`context`만 본다. → [[Hermes-Runtime-Internals]] §1

### 1.6 스킬 (`skills:`)

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `skills.creation_nudge_interval` | `15` | 주기적으로 워크플로의 skill화를 유도 | [[Hermes-Skills]] |
| `skills.config` | (미명시) | skill의 config 값 — 로드 시 컨텍스트에 자동 주입 | [[Hermes-Skills]] |

---

## 2. 게이트웨이 / 세션 (`config.yaml`)

### 2.1 플랫폼 툴셋 (`platform_toolsets:`)

| 플랫폼 | 전용 툴셋 | 출처 |
|--------|-----------|------|
| discord | `hermes-discord` | [[Hermes-Deployment-Surface]] |
| slack | `hermes-slack` | [[Hermes-Deployment-Surface]] |
| telegram | `hermes-telegram` | [[Hermes-Deployment-Surface]] |
| whatsapp | `hermes-whatsapp` | [[Hermes-Deployment-Surface]] |

### 2.2 세션 모델

| 키 | 기본값 | 의미 | 출처 |
|----|--------|------|------|
| `group_sessions_per_user` | `true` | 사용자별 세션 분리(대화·메모리 격리) | [[Hermes-Deployment-Surface]] |
| `max_concurrent_sessions` | `null` (무제한) | 동시 세션 상한 | [[Hermes-Deployment-Surface]] |
| `session_reset` | (미명시; `idle`/`at_hour` 기준) | 세션 리셋 정책 | [[Hermes-Deployment-Surface]] |

### 2.3 멀티 프로필

| 항목 | 값 | 의미 | 출처 |
|------|----|------|------|
| profile | (미명시) | 독립된 봇 토큰·세션·메모리·정체성 묶음. managed service(launchd/systemd)로 다중 가동 | [[Hermes-Deployment-Surface]] |

---

## 3. API 서버 (OpenAI 호환 백엔드)

| 항목 | 값 | 의미 | 출처 |
|------|----|------|------|
| 기본 포트 | `8642` (예시 `localhost:8642`) | API 서버 리스닝 포트 | [[Hermes-Deployment-Surface]] |
| 엔드포인트 | `/v1/chat/completions` 등 | OpenAI 호환 엔드포인트 | [[Hermes-Deployment-Surface]] |
| `API_SERVER_*` | (미명시) | CORS·API 키 등 API 서버 설정 환경변수 접두 | [[Hermes-Deployment-Surface]] |
| 인증 헤더 | `Authorization: Bearer <key>` | 요청 인증 | [[Hermes-Deployment-Surface]] |

> 에이전트 **전체**(터미널·파일·웹·메모리·스킬 툴셋)를 OpenAI 형식으로 노출. 스트리밍 시 tool progress가 inline. → [[Hermes-Deployment-Surface]] §1

---

## 4. 위임 / 오케스트레이션 (`claude -p` 호출 옵션)

> Hermes의 `terminal` 툴에서 `claude -p`(Claude Code 헤드리스)를 부를 때의 옵션. Hermes config가 아니라 위임 대상 CLI 플래그지만 추적 위해 수록.

| 플래그 | 값/예시 | 의미 | 출처 |
|--------|---------|------|------|
| `-p` / `--print` | — | 비대화형(헤드리스) 모드 | [[Orchestrating-Claude-Code]] |
| `--output-format` | `text` \| `json` \| `stream-json` | 출력 파싱 형식 | [[Orchestrating-Claude-Code]] |
| `--input-format` | `stream-json` | 입력 스트리밍(조합용) | [[Orchestrating-Claude-Code]] |
| `--include-partial-messages` | — | 부분 메시지 포함(스트리밍 조합) | [[Orchestrating-Claude-Code]] |
| `--permission-mode` | `acceptEdits` \| `auto` | 자동 승인 수준(`acceptEdits` 권장 출발점) | [[Orchestrating-Claude-Code]] |
| `--dangerously-skip-permissions` | — | 모든 검사 우회 — 격리 환경에서만 | [[Orchestrating-Claude-Code]] |
| `--model` | `claude-opus-4-8` (예시) | 위임 모델 지정 | [[Orchestrating-Claude-Code]] |
| `--max-turns` | `20` (예시) | 폭주 방지 턴 상한 | [[Orchestrating-Claude-Code]] |
| `--resume` | `<session-id>` | 기존 세션 이어가기 | [[Orchestrating-Claude-Code]] |
| `--fork-session` | — | 재개하며 새 세션 분기 | [[Orchestrating-Claude-Code]] |
| `--mcp-config` | `<json>` | 위임 받은 Claude에 추가 MCP 서버 연결 | [[Orchestrating-Claude-Code]] |

> 출력 JSON: `{ result, session_id, total_cost_usd, num_turns, ... }`. → [[Orchestrating-Claude-Code]] §4

### 4.1 ACP 어댑터 (참고)

| 항목 | 값 | 의미 | 출처 |
|------|----|------|------|
| copilot ACP 클라이언트 timeout | `900s` | `copilot --acp` 단명 세션 호출 한도 | [[Orchestrating-Claude-Code]] |
| 권한 브리지 | `allow_once` → `once`; 타임아웃/실패 시 기본 **deny** | ACP permission 매핑 | [[Orchestrating-Claude-Code]] |

---

## 5. 훅 계층 (Gateway / Plugin / Shell)

| 차원 | Shell hooks | Plugin hooks | Gateway hooks | 출처 |
|------|-------------|--------------|---------------|------|
| 선언 위치 | `config.yaml`의 `hooks:` 블록 | 플러그인 `register_hook()` | `HOOK.yaml` + `handler.py` 디렉터리 | [[Agent-Event-Hooks]] |
| 디렉터리 | `~/.hermes/agent-hooks/` | `~/.hermes/plugins/<name>/` | `~/.hermes/hooks/<name>/` | [[Agent-Event-Hooks]] |
| 언어 | 무엇이든(Bash/Python/Go…) | Python | Python | [[Agent-Event-Hooks]] |
| 실행 위치 | CLI + 게이트웨이 | CLI + 게이트웨이 | 게이트웨이 전용 | [[Agent-Event-Hooks]] |
| 툴 호출 차단 | ✅ `pre_tool_call` | ✅ | ❌ | [[Agent-Event-Hooks]] |
| LLM 컨텍스트 주입 | ✅ `pre_llm_call` | ✅ | ❌ | [[Agent-Event-Hooks]] |
| 프로세스 격리 | ✅ (서브프로세스) | ❌ (인프로세스) | ❌ | [[Agent-Event-Hooks]] |
| 동의 | `(event, command)`쌍 첫 사용 시 프롬프트 | 암묵(플러그인 신뢰) | 암묵(디렉터리 신뢰) | [[Agent-Event-Hooks]] |

### 5.1 Shell hook 이벤트

| 이벤트 | 시점 | 출처 |
|--------|------|------|
| `pre_tool_call` | 툴 실행 전(차단 가능) | [[Agent-Event-Hooks]] |
| `post` (tool) | 툴 실행 후(포맷·로깅·CI 트리거) | [[Agent-Event-Hooks]] |
| `pre_llm_call` | 다음 LLM 턴 전(컨텍스트 주입) | [[Agent-Event-Hooks]] |
| `subagent_stop` | 서브에이전트 종료 관측 | [[Agent-Event-Hooks]] |

### 5.2 Gateway hook 이벤트 (`HOOK.yaml`의 `events:`)

| 이벤트 | 발생 시점 | 주요 context | 출처 |
|--------|----------|-------------|------|
| `gateway:startup` | 게이트웨이 시작 | `platforms` | [[Agent-Event-Hooks]] |
| `session:start` / `:end` / `:reset` | 세션 생성/종료/`/new` | `platform`, `user_id`, `session_key` | [[Agent-Event-Hooks]] |
| `agent:start` | 메시지 처리 시작 | `message` | [[Agent-Event-Hooks]] |
| `agent:step` | 툴 호출 루프 매 반복 | `iteration`, `tool_names` | [[Agent-Event-Hooks]] |
| `agent:end` | 처리 완료 | `message`, `response` | [[Agent-Event-Hooks]] |
| `command:*` | 슬래시 명령 실행(와일드카드) | `command`, `args` | [[Agent-Event-Hooks]] |

> 핸들러 함수명은 반드시 `handle(event_type, context)` (async/def 둘 다 가능). 3계층 모두 non-blocking(훅 에러는 로깅, 에이전트 안 죽임). → [[Agent-Event-Hooks]] §3·§5

---

## 6. 메모리 / 세션

### 6.1 영속 메모리 파일

| 파일 | 위치 | 용량 한도 | 용도 | 출처 |
|------|------|-----------|------|------|
| `MEMORY.md` | `~/.hermes/memories/` | 2,200자 (~800토큰) | 에이전트 개인 노트(환경·관례·학습) | [[Agent-Persistent-Memory]] |
| `USER.md` | `~/.hermes/memories/` | 1,375자 (~500토큰) | 사용자 프로필(선호·스타일·기대) | [[Agent-Persistent-Memory]] |

| 동작 | 값/규칙 | 출처 |
|------|---------|------|
| 주입 방식 | 세션 시작 시 시스템 프롬프트에 **frozen snapshot**(세션 중 불변; 다음 세션부터 반영) | [[Agent-Persistent-Memory]] |
| `memory` 툴 액션 | `add` / `replace` / `remove` (read 없음) | [[Agent-Persistent-Memory]] |
| 한도 초과 | auto-compact 안 함 — **에러**를 던져 에이전트가 직접 큐레이션 | [[Agent-Persistent-Memory]] |
| 항목 구분자 | `§` | [[Agent-Persistent-Memory]] |
| 보조 검색 | `session_search`(과거 대화 RAG식 검색) | [[Agent-Persistent-Memory]] |

> frozen snapshot의 이유: 세션 중 시스템 프롬프트가 바뀌면 prefix cache가 깨지므로 고정. → [[Agent-Persistent-Memory]] §2

### 6.2 멀티테넌트 메모리(별도 아키텍처)

> Hermes config가 아니라 Hermes를 백엔드로 둔 멀티테넌트 KB 설계의 식별자·격리 파라미터. 추적 위해 수록.

| 항목 | 값 | 의미 | 출처 |
|------|----|------|------|
| `tenant_id` | `user:<id>` / `team:<id>` | 테넌트 격리 키 | [[Multi-Tenant-Memory-Agent]] |
| `scope` | `personal` \| `team` | 메모리 범위 | [[Multi-Tenant-Memory-Agent]] |
| `kind` | `fact` \| `preference` \| `episode` | 메모리 종류 | [[Multi-Tenant-Memory-Agent]] |
| `embedding` | `vector(1024)` (bge-m3) | 임베딩 차원 | [[Multi-Tenant-Memory-Agent]] |
| `importance` | `0.5` (기본) | 중요도 점수 | [[Multi-Tenant-Memory-Agent]] |
| RLS 세션 변수 | `app.tenant_id` | PostgreSQL Row-Level Security 필터 | [[Multi-Tenant-Memory-Agent]] |

---

## 7. 스킬 / 플러그인 / 디렉터리 레이아웃

### 7.1 디렉터리

| 경로 | 내용 | 출처 |
|------|------|------|
| `~/.hermes/skills/` | 모든 skill(번들·허브·자작) 단일 진실 소스 | [[Hermes-Skills]] |
| `~/.hermes/plugins/<name>/` | 커스텀 도구·훅 플러그인 | [[Hermes-Plugins]] |
| `~/.hermes/hooks/<name>/` | Gateway hooks | [[Agent-Event-Hooks]] |
| `~/.hermes/agent-hooks/` | Shell hooks | [[Agent-Event-Hooks]] |
| `~/.hermes/memories/` | `MEMORY.md` / `USER.md` | [[Agent-Persistent-Memory]] |
| `~/.hermes/.env` | 시크릿(메신저 표면에서 묻지 않고 여기/`hermes setup`로) | [[Hermes-Skills]] |

### 7.2 SKILL.md 메타데이터 키

| 키 | 의미 | 출처 |
|----|------|------|
| `name` | skill 식별자 | [[Hermes-Skills]] |
| `description` | Level 0 목록 노출 — 모델의 1차 선택 신호 | [[Hermes-Skills]] |
| `platforms` | OS 제한(예: `[macos, linux]`) | [[Hermes-Skills]] |
| `metadata.hermes.requires_toolsets` | 해당 툴셋 있을 때만 노출 | [[Hermes-Skills]] |
| `metadata.hermes.fallback_for_toolsets` | 해당 툴셋 있으면 숨김, 없으면 폴백 노출 | [[Hermes-Skills]] |
| `metadata.hermes.requires_tools` / `fallback_for_tools` | 위의 tool 단위 버전 | [[Hermes-Skills]] |
| `metadata.hermes.config` | `config.yaml` 연동(로드 시 주입) | [[Hermes-Skills]] |

> 예: 내장 `duckduckgo-search`는 `fallback_for_toolsets: [web]` — `FIRECRAWL_API_KEY` 있으면 web 툴셋 활성→DDG 숨김, 없으면 DDG 폴백 등장. → [[Hermes-Skills]] §5

### 7.3 플러그인 파일

| 파일 | 역할 | 출처 |
|------|------|------|
| `plugin.yaml` | 매니페스트(name·version·description) | [[Hermes-Plugins]] |
| `__init__.py` | `register(ctx)` 진입점(스키마↔핸들러 배선) | [[Hermes-Plugins]] |
| `schemas.py` | 툴 스키마(LLM이 보는 것) | [[Hermes-Plugins]] |
| `tools.py` | 툴 핸들러(실제 실행 코드) | [[Hermes-Plugins]] |

---

## 8. Cron / 예약 실행

| 항목 | 값/예시 | 의미 | 출처 |
|------|---------|------|------|
| `cronjob` 툴 action | `add`·`pause`·`resume`·`edit`·`trigger`·`remove` | 단일 툴로 전체 관리 | [[Agent-Scheduled-Execution]] |
| 스케줄 형식 | one-shot(`30m`) / 반복(`every 2h`) / cron식 / 자연어 | 예약 표현 | [[Agent-Scheduled-Execution]] |
| `--skill` | 예: `--skill blogwatcher` | 작업당 0~다수 skill 첨부 | [[Agent-Scheduled-Execution]] · [[Hermes-Skills]] |
| `--name` | 예: `--name "Skill combo"` | 작업 이름 | [[Agent-Scheduled-Execution]] |
| 실행 형태 | 기본 fresh agent 세션 / no-agent(스크립트만, LLM 미호출) | 모드 | [[Agent-Scheduled-Execution]] |
| 결과 전달 | 원 채팅 / 로컬 파일 / 플랫폼 타깃(Telegram·Slack 등) | 출력 대상 | [[Agent-Scheduled-Execution]] |
| runaway 방지 | cron 실행 세션 내에서 cron 관리 툴 **비활성화**(재귀 차단) | 안전장치 | [[Agent-Scheduled-Execution]] |
| 무인 인증 | `hermes model` provider 사용; OAuth 자동 갱신 방식 권장 | provider | [[Agent-Scheduled-Execution]] |

---

## 9. 환경변수 / CLI 명령

| 항목 | 값 | 의미 | 출처 |
|------|----|------|------|
| `FIRECRAWL_API_KEY` | (시크릿) | 있으면 `web` 툴셋 활성(없으면 DDG 폴백) | [[Hermes-Skills]] |
| `API_SERVER_*` | (미명시) | API 서버 CORS·키 설정 환경변수 접두 | [[Hermes-Deployment-Surface]] |
| `hermes setup` | — | 시크릿 설정 CLI | [[Hermes-Skills]] |
| `hermes cron create ...` | — | cron 생성 CLI | [[Agent-Scheduled-Execution]] |
| `hermes model` | — | 무인 실행 provider 선택 | [[Agent-Scheduled-Execution]] |
| `hermes acp` / `hermes-acp` | — | Hermes를 ACP 서버로 기동(에디터가 붙음) | [[Orchestrating-Claude-Code]] |

---

## 10. 관련
- [[Hermes-Runtime-Internals]] — terminal·compression·guardrails·delegation 기본값 원천
- [[Hermes-Deployment-Surface]] — API 서버·게이트웨이·세션·멀티프로필
- [[Hermes-Skills]] — SKILL.md 메타·조건부 활성·skills.config
- [[Hermes-Plugins]] — plugin 디렉터리·register(ctx)
- [[Agent-Event-Hooks]] — 3계층 훅·이벤트·디렉터리
- [[Agent-Persistent-Memory]] — MEMORY.md/USER.md 한도·frozen snapshot
- [[Agent-Scheduled-Execution]] — cronjob 툴·no-agent·runaway 방지
- [[Orchestrating-Claude-Code]] — `claude -p` 위임 플래그·ACP·타임아웃
- [[Multi-Tenant-Memory-Agent]] — 멀티테넌트 KB 식별자·RLS
- [[Hermes-Agent]] — function calling 기본(개념)
