---
tags:
  - ai
  - claude-code
  - headless
  - automation
  - orchestration
  - agent
created: 2026-06-24
---

# Claude Code 헤드리스 모드 (`claude -p`)

> [!summary] 한 줄 요약
> `claude -p`(=`--print`)는 Claude Code를 **비대화형(헤드리스)**으로 1회 실행해 결과를 stdout으로 회수하는 모드. 스크립트·서버·오케스트레이터가 Claude Code를 **서브프로세스로 호출**하는 표준 방법이며, **로그인된 구독 인증을 재사용**하므로 별도 API 키·종량 과금 없이 쓸 수 있다(사용량은 구독 한도에서 차감). Hermes 서브에이전트 위임([[Orchestrating-Claude-Code]])의 공통 기반.

---

## 1. 왜 헤드리스인가

```
대화형 TUI            → 사람이 터미널에서 직접 대화
헤드리스(claude -p)   → 프로그램이 1회 호출 → stdout 회수 → 파싱 → 다음 단계
```
- **자동화·통합**: 쉘/파이썬/서버가 `claude -p "..."`로 호출, 결과를 받아 후속 처리.
- **구독 재사용·무과금**: Anthropic API(키·종량 과금) 대신 **로그인된 Claude Code 구독**으로 인증 → 추가 청구 0. (단 사용량은 구독 롤링/주간 한도에서 차감 → §5)
- **오케스트레이션 기반**: 로컬·저렴 모델을 플래너/라우터로 두고 무거운 작업만 Claude에 위임하는 계층의 실행 채널.

---

## 2. 핵심 옵션

```bash
claude -p "리팩터링: src/foo.py 중복 제거" \
  --output-format json \           # 결과 형식: text | json | stream-json
  --permission-mode acceptEdits \  # ★ 무인 자동화 필수 (없으면 권한 프롬프트에서 hang)
  --model claude-opus-4-8 \        # 모델 지정
  --max-turns 20                   # 에이전트 턴 상한 (폭주 방지)
```

| 옵션 | 용도 |
|---|---|
| `-p`, `--print` | 비대화형 1회 실행 후 종료 (헤드리스 진입점) |
| `--output-format <text\|json\|stream-json>` | `json`=단일 결과 객체, `stream-json`=실시간 청크(NDJSON) |
| `--input-format <text\|stream-json>` | 입력 형식. stream-json 입력으로 멀티턴 프로그램 공급 |
| `--include-partial-messages` | stream-json에 부분 토큰 청크 포함(진행 가시화) |
| `--permission-mode <mode>` | `default`/`acceptEdits`/`plan`/`bypassPermissions` (§3) |
| `--dangerously-skip-permissions` | 모든 권한 검사 우회 ⚠️ 격리 환경 전용 |
| `--allowedTools` / `--disallowedTools` | 허용·차단 툴 화이트/블랙리스트 |
| `--model <id>` | 예: `claude-opus-4-8`, `sonnet`, `haiku` |
| `--max-turns <n>` | 내부 에이전트 루프 상한 |
| `--resume <session-id>` / `--continue` | 기존 세션 이어가기(멀티턴 위임) |
| `--fork-session` | 재개하되 새 세션으로 분기(원본 보존) |
| `--mcp-config <json\|file>` | 위임받은 Claude에 추가 MCP 서버 주입 |
| `--append-system-prompt <text>` | 시스템 프롬프트 추가(역할·제약 주입) |
| `--add-dir <path>` | 작업 허용 디렉터리 추가 |

---

## 3. 권한 모드 — 자동화의 핵심 함정 ⚠️

```
대화형 기본값으로 -p 호출 시 권한 프롬프트에서 멈춤 → 무인 호출이 행(hang)
  default            : 매 위험 동작마다 확인 (무인 부적합)
  acceptEdits        : 파일 편집 자동 승인 (무인 출발점, 권장)
  plan               : 계획만 세우고 실행 안 함(읽기 전용 탐색)
  bypassPermissions  : 광범위 자동 승인
  --dangerously-skip-permissions : 모든 검사 우회
```
> [!danger] `--dangerously-skip-permissions` / `bypassPermissions`는 임의 명령을 무확인 실행한다. **반드시 샌드박스·전용 작업디렉터리·컨테이너 안에서만.** → [[Agent-Security]]

---

## 4. 결과 회수 (JSON)

```bash
claude -p "..." --output-format json
```
```jsonc
{
  "type": "result",
  "result": "…최종 응답 텍스트…",
  "session_id": "…",          // --resume 으로 이어갈 키
  "total_cost_usd": 0.019,     // API 환산 비용(구독이면 참고치)
  "num_turns": 3,
  "duration_ms": 4300,
  "is_error": false,
  "usage": { "input_tokens": …, "output_tokens": … }
}
```
- 오케스트레이터가 `result`를 받아 검증→다음 프롬프트 생성, 실패 시 같은 `session_id`로 `--resume` 하는 **피드백 루프** 구성.
- `stream-json`은 진행 중 청크를 NDJSON으로 흘려 장시간 작업의 진행 표시·조기 판단에 유리.

---

## 5. 비용·한도 ⚠️

- **API 비용 0**: API 키가 아니라 **로그인된 Claude Code 구독**으로 인증 → 별도 청구 없음.
- **단, 구독 한도에서 차감**: 모든 `claude -p` 호출이 **롤링 5시간 + 주간 한도**를 소비. 캐시 입력도 처리량으로 집계됨.
- **병렬 펼침 주의**: 서브에이전트를 많이 병렬 호출하면 비용이 아니라 **rate limit(한도)에 먼저 막힘**. → 라우팅·반복 판단은 로컬(gemma)에 두고 Claude 위임은 고난도 건만으로 절약. 한도는 서버에서 `claude` 실행 후 `/usage`로 확인.

---

## 6. 타임아웃 함정 (오케스트레이터 측) ⚠️

```
호출자(Hermes 등)의 쉘/실행 타임아웃에 긴 Claude 작업이 잘릴 수 있음
  예) ~/.hermes: terminal.timeout 180s / code_execution.timeout 300s
대응: ① 작업을 작게 쪼개 위임  ② 호출자 timeout 상향
      ③ 백그라운드 실행 후 폴링  ④ stream-json으로 진행 가시화
```

---

## 7. 활용 예

| 용도 | 호출 형태 | 노트 |
|---|---|---|
| RAG 분류·라우팅 엔진 | `claude -p --model sonnet --effort low --output-format json` | [[Orchestrating-Claude-Code]] |
| Hermes→Claude Code 서브에이전트 위임 | Hermes `terminal` 툴 → `claude -p ... --permission-mode acceptEdits` | [[Orchestrating-Claude-Code]] |
| 무인 자동 실행(cron·CI) | 위 + 샌드박스·권한 모드 | [[Agent-Scheduled-Execution]] · [[Agent-Security]] |

> 단순 분류·라우팅처럼 "인덱스 배열만 반환"하는 용도는 `--model sonnet --effort low`가 스윗스팟(haiku는 effort 미지원·출력 장황).

---

## 8. 관련
- [[Orchestrating-Claude-Code]] — 오케스트레이터→Claude Code 위임(CLI 헤드리스/ACP/MCP)
- [[Agent-Security]] — 무인 위임 시 권한·샌드박싱 통제
- [[Agent-Scheduled-Execution]] — cron 자율 실행에서 헤드리스 호출
- [[Multi-Agent-Coding-Team]] — 오케스트레이터=검증자, Claude=실행자 역할 분담
- [[Prompt-Caching]] — 반복 호출 시 프롬프트 캐시로 입력 비용 절감
