---
tags:
  - ai
  - agent
  - hermes
  - orchestration
  - examples
created: 2026-06-24
---

# Hermes 실사용 예시 모음 (에이전트 오케스트레이션)

> [!summary] 한 줄 요약
> Hermes 에이전트 오케스트레이션을 **실제로 어떻게 쓰는지** 시나리오별 구체 예시로 정리한다. 각 예시는 **시나리오 → 구성 → 명령/설정 스니펫 → 핵심 포인트** 순. 명령·플래그·설정은 근거 노트에 등장한 것만 사용했고, 여러 메커니즘을 묶은 것은 **(구성안)**, 이 볼트에서 실측·운용한 것은 **(실운용)**으로 표기한다.

> [!important] 표기 규칙
> - **(실운용)** — 이 볼트에서 실제 측정·운영한 사례. 예시 5(RAG 라우팅)만 해당.
> - **(구성안)** — 문서화된 개별 메커니즘을 조합한 시나리오. 조합 자체는 실측 아님.
> - 명령·플래그가 어느 노트에서 왔는지는 각 예시 끝 "근거"에 표시.

---

## 예시 1 — Hermes 오케스트레이터 → Claude Code 위임 (구성안)

**시나리오.** 로컬·무료 모델(Hermes/Gemma)을 플래너·라우터·검증자로 두고, 무거운 코딩 작업만 Claude Code(Opus)에 위임한다. 라우팅·반복 판단은 로컬에서 싸게, 고난도 편집만 비싼 모델에.

**구성.**

```
사용자 → 오케스트레이터(Hermes/Gemma, 로컬·무료·빠름)   ← 플래너/라우터/검증
            └─ 위임: claude -p (Opus, 고품질)            ← 실제 코딩 실행
            └─ 결과(JSON) 받아 검증 → 다음 단계 결정
```

Hermes는 `terminal` 툴로 임의 쉘 명령을 실행할 수 있으므로, 그냥 `claude -p "..."`를 호출하면 위임이 성립한다. Claude Code는 `-p/--print` 비대화형(헤드리스) 모드라 stdout으로 결과를 회수한다.

**명령 스니펫 — CLI 헤드리스 호출.**

```bash
claude -p "리팩터링: src/foo.py 중복 제거" \
  --output-format json \           # text | json | stream-json
  --permission-mode acceptEdits \  # 무인 자동화 필수 (없으면 권한 프롬프트에서 hang)
  --model claude-opus-4-8 \
  --max-turns 20                   # 폭주 방지 상한
```

**결과 회수 → 검증 → 피드백 루프.**

```jsonc
// claude -p ... --output-format json 의 반환
{
  "type": "result",
  "result": "…최종 응답 텍스트…",
  "session_id": "…",          // --resume 으로 이어갈 키
  "total_cost_usd": 0.019,
  "num_turns": 3,
  "is_error": false
}
```

```
오케스트레이터(Gemma)가 result를 받아 → 테스트 실행·검증 → 다음 프롬프트 생성
실패 시 같은 session_id 로 --resume 하여 수정 위임 (멀티턴 피드백 루프)
  claude -p "테스트 X가 실패한다, 수정해라" --resume <session-id> --output-format json
  --fork-session : 재개하되 새 세션으로 분기(원본 보존)
```

**핵심 포인트.**
- 연결 방식 3가지 중 **CLI 헤드리스가 기본**(단순·견고·의존성 0). ACP는 에디터 통합용이고 방향이 반대(에디터가 Hermes에 붙는 서버)라 단순 위임엔 과하다. MCP는 Claude에 우리 도구를 쥐여줄 때(`--mcp-config`).
- **권한 모드 함정**: 기본(대화형)으로 호출하면 권한 프롬프트에서 멈춰(hang) 무인 위임이 죽는다 → `--permission-mode acceptEdits`가 출발점.
- **타임아웃 함정**: 호출자(Hermes) 측 `terminal.timeout: 180`, `code_execution.timeout: 300`에서 긴 작업이 잘린다 → 작업 잘게 쪼개기 / timeout 상향 / 백그라운드 폴링 / stream-json 진행 가시화.

근거: [[Orchestrating-Claude-Code]] §3·§4 (호출·피드백 루프), [[Claude-Code-Headless]] §2~4 (옵션·JSON 결과)

---

## 예시 2 — 무인 cron 자율 실행 (구성안)

**시나리오.** 사람 없이 일정에 따라 에이전트를 깨워 작업시킨다. 매시간 피드 요약, 매일 헬스체크 같은 정기 작업.

**구성 — 단일 `cronjob` 툴, 자연어로 관리.**

```bash
# 자연어/CLI 어느 쪽으로도 등록 (별도 schedule/list/remove 툴 없이 cronjob 하나가 action으로 처리)
/cron add 30m "빌드 확인 리마인드"
/cron add "every 2h" "서버 상태 체크"
/cron add "every 1h" "새 피드 요약" --skill blogwatcher
hermes cron create "every 1h" "..." --skill blogwatcher --skill maps --name "Skill combo"
```

**비용 절감 — no-agent 모드.**

```
추론이 필요 없으면 LLM을 호출하지 마라.
  no-agent 모드: 스케줄에 "스크립트"만 걸고 LLM은 전혀 안 부름 → stdout 그대로 전달. 비용 0·결정론적
  언제: 디스크 사용량 보고·헬스체크처럼 판단이 불필요한 정기 작업
  대비: fresh-session 모드(LLM 새 세션)는 판단·도구사용이 필요한 작업에만
```

**무인 안전장치(이것이 핵심).**

```
방어1: 예약 실행 세션 안에서는 cron 관리 툴이 비활성화됨 → 예약이 예약을 못 낳음(재귀 차단)
방어2: tool-loop 가드레일 — 무진전 반복을 카운터로 차단 (무인이면 hard_stop_enabled: true 고려)
방어3: 권한 모드 + 터미널 샌드박스 — 행동 범위 제한
인증 : cron은 hermes model 의 provider 사용. 무인 권장 = OAuth 자동 갱신(한밤중 토큰 만료로 실패 방지)
```

위임(예시 1)과 결합하면, cron 세션이 `claude -p ... --permission-mode acceptEdits`를 격리 작업디렉터리에서 호출하는 무인 코딩 파이프라인이 된다. 이때 `--dangerously-skip-permissions` / `bypassPermissions`는 **반드시 샌드박스 안에서만**.

**핵심 포인트.**
- 무인 자율의 제1 위험은 통제 없는 자기증식 → Hermes는 cron 실행 컨텍스트에서 cron 생성 툴을 끈다.
- 가드레일 기본값은 보수적(`hard_stop_enabled: false`)이라 정당한 재시도를 안 끊는다. **무인 환경이라면 hard stop을 켜는 걸 고려**.
- 결과 전달: 원 채팅 / 로컬 파일 / 설정된 플랫폼 타깃(Telegram·Slack 등) → 예시 3의 게이트웨이로 연결.

근거: [[Agent-Scheduled-Execution]] §1~5 (cron 툴·no-agent·runaway 방지·인증), [[Hermes-Runtime-Internals]] §3·§4 (가드레일·샌드박스), [[Claude-Code-Headless]] §3 (권한 모드)

---

## 예시 3 — 멀티플랫폼 게이트웨이 봇 (구성안)

**시나리오.** 개발(CLI)·위임(orchestration)을 넘어, 툴 장착 에이전트를 메신저로 상시 노출한다 — Telegram/Discord/Slack/WhatsApp 봇 또는 웹 UI 백엔드.

**구성 — 두 표면.**

```
① API 서버 (OpenAI 호환 백엔드)
   hermes를 OpenAI 형식 HTTP 엔드포인트로 노출 → Open WebUI·LobeChat·LibreChat 등이 백엔드로 사용
   요청은 에이전트 "풀 툴셋"(terminal·file·web·memory·skills)으로 처리 후 응답
```
```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer <key>" -d '{...}'
# CORS·키 설정(API_SERVER_*), /v1/chat/completions 엔드포인트
```
```
② 멀티플랫폼 게이트웨이 (메신저 봇으로 상주)
   config platform_toolsets: 플랫폼별 전용 툴셋
     discord → hermes-discord / slack → hermes-slack / telegram → hermes-telegram / whatsapp → hermes-whatsapp
   gateway hooks가 이 표면에서 발화(로깅·알림·웹훅)
```

**세션 모델.**

```
group_sessions_per_user: true   — 사용자별 세션 분리(대화·메모리 격리)
max_concurrent_sessions: null   — 동시 세션 상한(기본 무제한)
session_reset: idle / at_hour 기준 리셋
→ 플랫폼·user별로 세션이 갈리고 각 세션이 영속 메모리를 가짐
```

**핵심 포인트.**
- [[Local-LLM-API-Deployment]]의 raw 모델 API와 구분: 여기 API 서버는 **모델이 아니라 에이전트 전체**(툴·메모리·스킬)를 노출한다.
- 스트리밍 시 tool progress가 inline으로 흘러 프론트가 "에이전트가 뭐 하는지" 표시할 수 있다.
- cron(예시 2) 결과를 게이트웨이 타깃으로 전달하면 "상시 가동 어시스턴트"가 완성된다.

근거: [[Hermes-Deployment-Surface]] §1~3 (API 서버·게이트웨이·세션)

---

## 예시 4 — 멀티 프로필 Kanban 협업 (구성안)

**시나리오.** 여러 named 봇(`inbox-triage`, `ops-review` 등)이 분업해 협업한다. 작업이 에이전트 경계를 넘고, 재시작을 견디고, 사람 개입이 가능해야 할 때 — `delegate_task`(휘발) 대신 durable 보드.

**구성 — 두 축의 결합.**

```
멀티 프로필 (배포 표면):
  profile = 독립된 봇 토큰·세션·메모리·정체성 묶음
  한 머신에서 여러 프로필을 managed service(launchd/systemd)로 동시 가동
Kanban (조율 보드):
  named 프로필들이 영속 SQLite 보드(~/.hermes/kanban.db)로 협업
  작업=row, 핸드오프=row → 다른 프로필·사람 누구나 R/W
```

**두 표면 — 모델은 툴, 사람은 CLI.**

```
모델 : kanban_* 툴셋으로 직접 — kanban_show/list/create/complete/block/unblock/heartbeat/comment/link
       (dispatcher가 worker를 이 툴들과 함께 스폰. shell로 `hermes kanban` 호출 ❌)
사람·cron·대시보드 : `hermes kanban …` / `/kanban` / 웹 대시보드
→ 둘 다 같은 kanban_db 레이어 → 읽기 일관·쓰기 드리프트 없음
```

**Dispatcher·생존성.**

```
Dispatcher: 게이트웨이 안 long-lived 루프(기본 60초)
  ① stale 클레임 회수 ② crash worker 회수 ③ ready 작업 승격 ④ atomic claim 후 프로필 스폰
  연속 스폰 실패 kanban.failure_limit(기본 2)회 → 자동 block (thrashing 방지)
생존성: 긴 작업은 kanban_heartbeat(note=...)로 생존 신호
  dispatch_stale_timeout(기본 4h) 경과 + 최근 1h heartbeat 없음 → crash 간주 → reclaim(작업이 ready로 복귀)
```

**핵심 포인트.**
- `delegate_task` vs Kanban: 전자는 **함수 호출(in-process·동기·휘발)**, 후자는 **작업 큐(durable·peer·재개)**. 짧은 하위추론이면 delegate_task, 살아남아야 하면 Kanban.
- crash 복원력이 핵심 — 프로세스가 죽어도 작업은 보드에 남아 다른 worker가 이어받는다.
- cron(예시 2)이 Kanban 작업을 생성·트리거하면 "매일 브리프가 몇 주에 걸쳐 저널 축적" 같은 장기 ops가 된다.

근거: [[Agent-Kanban]] §1~6 (프리미티브·툴·dispatcher·생존성), [[Hermes-Deployment-Surface]] §4 (멀티 프로필), [[Hermes-Runtime-Internals]] §1 (delegate_task 대비)

---

## 예시 비교 정리

| # | 예시 | 분류 | 핵심 메커니즘 | 안전·비용 레버 |
|---|---|---|---|---|
| 1 | Hermes → Claude Code 위임 | 구성안 | `claude -p` CLI 헤드리스 + `--resume` 루프 | 권한 모드·타임아웃 |
| 2 | 무인 cron 자율 실행 | 구성안 | `cronjob` 툴 + no-agent 모드 | 재귀 차단·가드레일·샌드박스 |
| 3 | 멀티플랫폼 게이트웨이 봇 | 구성안 | OpenAI 호환 API 서버 + platform_toolsets | per-user 세션 격리 |
| 4 | 멀티 프로필 Kanban 협업 | 구성안 | named 프로필 + durable SQLite 보드 | heartbeat·reclaim·failure_limit |

---

## 관련 노트
- [[Orchestrating-Claude-Code]] — 오케스트레이터→Claude Code 위임(CLI/ACP/MCP)
- [[Claude-Code-Headless]] — `claude -p` 옵션·JSON 결과·구독 재사용
- [[Hermes-Deployment-Surface]] — API 서버·멀티플랫폼 게이트웨이·멀티 프로필
- [[Hermes-Runtime-Internals]] — delegate_task·압축·가드레일·샌드박스
- [[Agent-Scheduled-Execution]] — cron 자율 실행·no-agent·runaway 방지
- [[Agent-Kanban]] — durable 멀티 에이전트 조율 보드
- [[Agent-Security]] — 무인 실행 권한·샌드박싱·승인 게이트
