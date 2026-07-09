---
tags:
  - ai
  - agent
  - hermes
  - hooks
  - extensibility
created: 2026-06-19
---

# 에이전트 이벤트 훅 (Event Hooks)

> [!summary] 한 줄 요약
> 에이전트의 **생명주기 지점**(시작·매 스텝·툴 호출 전후·세션 시작/종료)에 커스텀 코드를 끼워 넣는 확장점. Hermes는 **3계층 훅**(Gateway / Plugin / Shell)을 제공 — 로깅·알림은 물론 **위험 툴 호출 차단·자동 포맷·LLM 컨텍스트 주입**까지. Claude Code의 hooks와 동형 개념.

> 일반 가드레일은 [[Agent-Harness]]·[[Hermes-Runtime-Internals]], 차단·감사 관점은 [[Agent-Security]]. 이 노트는 그 통제를 **어디서 어떻게 끼우나**.

---

## 1. 왜 훅인가

```
에이전트 코드를 안 고치고 동작을 바꾸려면 = 생명주기 이벤트에 핸들러를 건다
  관측: 활동 로깅·메트릭·알림(긴 작업 시 텔레그램 알림 등)
  통제: 위험 명령 차단(pre_tool_call), 파괴적 write/patch 승인 요구
  자동화: 툴 실행 후 자동 포맷(post), CI 트리거
  증강: 다음 LLM 턴에 git status·날짜·문서 주입(pre_llm_call)
```

## 2. 3계층 훅 — 무엇을 어디에

| 차원 | **Shell hooks** | **Plugin hooks** | **Gateway hooks** |
|------|-----------------|------------------|-------------------|
| 선언 | `config.yaml`의 `hooks:` 블록 | 플러그인 `register_hook()` | `HOOK.yaml`+`handler.py` 디렉터리 |
| 위치 | `~/.hermes/agent-hooks/` | `~/.hermes/plugins/<name>/` | `~/.hermes/hooks/<name>/` |
| 언어 | **무엇이든**(Bash·Python·Go 바이너리…) | Python | Python |
| 실행 위치 | CLI + 게이트웨이 | CLI + 게이트웨이 | **게이트웨이 전용** |
| 툴 호출 차단 | ✅ (`pre_tool_call`) | ✅ | ❌ |
| LLM 컨텍스트 주입 | ✅ (`pre_llm_call`) | ✅ | ❌ |
| 동의(consent) | `(event, command)`쌍 첫 사용 시 프롬프트 | 암묵(플러그인 신뢰) | 암묵(디렉터리 신뢰) |
| 프로세스 격리 | ✅ (서브프로세스) | ❌ (인프로세스) | ❌ |

```
드롭인 단일 스크립트로 빠르게  → Shell hooks(언어 자유·서브프로세스 격리)
재사용 가능한 Python 확장      → Plugin hooks
메신저 게이트웨이 운영 이벤트  → Gateway hooks
```

## 3. Gateway 훅 — 형식

```text
~/.hermes/hooks/my-hook/
├── HOOK.yaml      # 구독할 이벤트 선언
└── handler.py     # async def handle(event_type, context)
```
```yaml
# HOOK.yaml
name: long-task-alert
events: [agent:start, agent:end, agent:step]   # command:* 같은 와일드카드 가능
```
```python
async def handle(event_type: str, context: dict):  # 이름은 반드시 handle
    ...   # async/def 둘 다 가능
```

### 이벤트 모델

| 이벤트 | 발생 시점 | 주요 context |
|--------|----------|-------------|
| `gateway:startup` | 게이트웨이 시작 | `platforms` |
| `session:start` / `:end` / `:reset` | 세션 생성/종료/`/new` | `platform`,`user_id`,`session_key` |
| `agent:start` | 메시지 처리 시작 | `message` |
| `agent:step` | **툴 호출 루프 매 반복** | `iteration`,`tool_names` |
| `agent:end` | 처리 완료 | `message`,`response` |
| `command:*` | 슬래시 명령 실행 | `command`,`args` |

- **와일드카드**: `command:*` 하나로 모든 슬래시 명령 감시.
- `agent:step`은 [[Hermes-Runtime-Internals]] tool-loop와 같은 지점 — 여기서 무한루프·비용을 관측·차단.

## 4. Shell 훅 — 차단·포맷·주입의 드롭인

```
pre_tool_call : 툴 실행 "전" — 위험 terminal 명령 reject, 디렉터리 정책 강제,
                파괴적 write_file/patch 승인 요구 → 차단 가능
post(tool)    : 툴 실행 "후" — 방금 쓴 .py/.ts 자동 포맷, API 호출 로깅, CI 트리거
pre_llm_call  : 다음 LLM 턴 "전" — git status·요일·검색 문서를 사용자 메시지에 prepend
subagent_stop : 서브에이전트 종료 관측 (→ [[Hermes-Runtime-Internals]] delegate_task)
```
- **서브프로세스 격리**: 셰방(shebang)만 있으면 어떤 언어든. 에이전트 인프로세스와 분리 → 크래시 전파 없음.
- **동의 모델**: `(event, command)` 쌍마다 첫 사용 시 승인 프롬프트 → 무단 차단/주입 방지.

## 5. non-blocking 안전성 ⚠️

```
3계층 모두 non-blocking: 훅 내부 에러는 잡아서 로깅, 에이전트를 죽이지 않음
  → 훅이 깨져도 본 파이프라인은 계속(관측·부가기능이 본체를 인질로 잡지 않음)
단, pre_tool_call의 "차단"은 의도된 흐름 제어(에러 아님) — 명시적 reject로 동작
```

## 6. 훅 vs 가드레일 vs 보안 — 역할 구분

```
훅(Hooks)        = 확장점(어디에 코드를 끼우나) — 메커니즘
가드레일(Guardrails) = 무한루프·실패 반복 차단 정책 ([[Hermes-Runtime-Internals]] §3)
보안(Security)    = 최소권한·샌드박싱·승인 게이트 정책 ([[Agent-Security]])
→ 훅은 가드레일·보안 정책을 "구현하는 자리": pre_tool_call 훅으로 위험 명령 차단 등
```

---

## 7. 관련
- [[Hermes-Runtime-Internals]] — tool-loop 가드레일·서브에이전트(agent:step·subagent_stop 지점)
- [[Agent-Security]] — pre_tool_call 차단·감사로 구현하는 보안 정책
- [[Agent-Harness]] — 에이전트 루프·확장점 일반 이론
- [[Hermes-Plugins]] — 플러그인이 register(ctx)로 훅을 등록(plugin hooks의 상위 시스템)
- [[MCP-Skill]] — 플러그인·도구 확장(훅과 함께 쓰는 확장 메커니즘)
- [[Tracing]] — 라이프사이클 이벤트를 추적·관측에 연계
- [[Hermes-Deployment-Surface]] — gateway hooks가 발화하는 게이트웨이 표면
- [[AOP-Logging]] — (대비) Spring AOP 기반 횡단 관심사
