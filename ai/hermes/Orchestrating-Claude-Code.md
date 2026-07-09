---
tags:
  - ai
  - agent
  - orchestration
  - claude-code
  - hermes
created: 2026-06-19
---

# 오케스트레이터 LLM으로 Claude Code 위임 실행 (Hermes → Claude Code)

> [!summary] 한 줄 요약
> **로컬·저렴한 모델(Hermes/Gemma)을 플래너·라우터**로 두고, **무거운 코딩 작업만 Claude Code(Opus)에 위임**하는 패턴. 연결 방식은 3가지 — ① `claude -p` **CLI 헤드리스**(가장 단순·견고), ② **ACP**(에디터 프로토콜, 방향 주의), ③ **MCP**(도구 노출). 실전 기본값은 ①.

> 이 노트는 로컬 환경([[Local-LLM-API-Deployment]]의 Hermes + `~/.hermes`)에서 실제 확인한 구성 기준. 런타임 일반 비교는 [[Autonomous-Agent-Runtimes]], 역할 분담은 [[Multi-Agent-Coding-Team]].

---

## 1. 왜 이 구조인가 — 계층 분리

```
사용자 → 오케스트레이터(Hermes/Gemma, 로컬·무료·빠름)   ← 플래너/라우터/검증
            └─ 위임: Claude Code(Opus, 유료·고품질)        ← 실제 코딩 실행
            └─ 결과 받아 검증 → 다음 단계 결정

이점: 라우팅·반복 판단은 로컬에서 싸게, 고난도 편집만 비싼 모델에.
      모델(지능)과 런타임(실행 루프)을 분리 → [[Autonomous-Agent-Runtimes]]
```
- Hermes는 `terminal` 툴로 임의 쉘 명령을 실행할 수 있어, **그냥 `claude -p "..."`를 호출**하면 위임이 성립한다.
- Claude Code는 `-p/--print` **비대화형(헤드리스) 모드**를 지원 → stdout으로 결과 회수.

---

## 2. 연결 방식 3가지

| 방식 | 메커니즘 | 장점 | 단점 | 추천 |
|------|----------|------|------|------|
| **CLI 헤드리스** | Hermes `terminal` 툴 → `claude -p` | 단순·견고, 의존성 0, JSON 파싱 | 프로세스 1회성(세션은 `--resume`) | ⭐ 기본 |
| **ACP** | Agent Client Protocol(stdio JSON-RPC) | 스트리밍·승인·diff 구조화 | **방향 주의**(아래 §5) | 에디터 통합 |
| **MCP** | Claude Code를 MCP 서버/클라이언트로 | 도구 단위 연동 | 오케스트레이션보단 도구 노출용 | 도구 공유 |

---

## 3. CLI 헤드리스 — 실전 호출 (권장)

```bash
claude -p "리팩터링: src/foo.py 중복 제거" \
  --output-format json \           # text | json | stream-json
  --permission-mode acceptEdits \  # 비대화 자동화: 편집 자동 승인
  --model claude-opus-4-8 \
  --max-turns 20                   # 폭주 방지 상한
```
- **출력 파싱**: `--output-format json`(단일 결과) 또는 `stream-json`(실시간 청크, `--input-format stream-json`·`--include-partial-messages`와 조합) → 오케스트레이터가 안정적으로 파싱.
- **세션 이어가기**: `--resume <session-id>` 또는 `--fork-session`(재개하며 새 세션 분기) → 멀티턴 위임 가능.
- **도구 주입**: `--mcp-config <json>`으로 위임 받은 Claude에 추가 MCP 서버 연결.

### 3.1 권한 모드 — 자동화의 핵심 함정 ⚠️

```
기본(대화형)으로 호출하면 권한 프롬프트에서 멈춤 → 무인 위임이 행(hang)
  --permission-mode acceptEdits : 파일 편집 자동 승인 (권장 출발점)
  --permission-mode auto        : 더 넓은 자동 승인
  --dangerously-skip-permissions: 모든 검사 우회 ⚠️ 격리 환경에서만
```
> [!danger] `--dangerously-skip-permissions`는 임의 명령을 무확인 실행한다. **반드시 샌드박스/격리(컨테이너·전용 작업디렉터리) 안에서만.** → [[Agent-Security]]

### 3.2 타임아웃 함정 ⚠️

```
Hermes 측 한도(~/.hermes/config.yaml):
  terminal.timeout: 180         # 쉘 명령 180초
  code_execution.timeout: 300   # 코드 실행 300초
→ 긴 Claude 작업은 이 한도에서 잘림(중간 종료)
대응: ① 작업을 작게 쪼개 위임  ② Hermes 측 timeout 상향
      ③ 백그라운드 실행 후 폴링  ④ 스트리밍으로 진행 가시화
```

---

## 4. 결과를 다시 오케스트레이터로

```
claude -p ... --output-format json
  → { result, session_id, total_cost_usd, num_turns, ... } 형태
  → Hermes(Gemma)가 result를 받아: 테스트 실행·검증·다음 프롬프트 생성
  → 실패 시 같은 session_id로 --resume 하여 수정 위임 (피드백 루프)
```
- 이 검증·재위임 루프가 [[Multi-Agent-Coding-Team]]의 QA 역할에 해당(오케스트레이터가 객관적 검증자).

---

## 5. ACP 방향 주의 — 흔한 오해 ⚠️

> [!warning] Hermes의 ACP는 "에디터가 Hermes에 붙는" **서버** 방향이다
> `hermes acp` / `hermes-acp`는 Hermes를 **ACP 서버**로 띄워 VS Code·Zed 같은 에디터가 stdio로 붙는 용도(`acp_adapter/server.py`, `hermes-acp` 툴셋). **Hermes가 Claude를 부르는 방향이 아니다.**

```
ACP로 Hermes → Claude Code를 하려면:
  Hermes가 ACP "클라이언트"가 되고 Claude Code가 ACP "서버"여야 함
  선례: Hermes는 copilot을 ACP 서버로 붙이는 클라이언트 어댑터를 가짐
        (agent/copilot_acp_client.py — `copilot --acp`를 단명 세션으로 호출, timeout 900s)
  → 동형으로 Claude Code의 ACP 서버(예: Zed의 claude-code-acp 어댑터)에 붙일 수 있음
정리: 단순 위임이면 ACP는 과함. CLI 헤드리스(§3)가 더 단순·견고.
      ACP는 스트리밍·diff·승인 UI가 필요한 에디터 통합에서 값어치.
```

### ACP 어댑터 내부 (참고)
- 동기 `AIAgent`를 async JSON-RPC stdio 서버로 감쌈. **stdout=프로토콜 전용, 로그는 stderr.**
- 세션 관리(`SessionManager`): create/fork/resume/cancel, cwd 바인딩(에디터 워크스페이스 기준).
- 권한 브리지: 위험 명령 승인 → ACP permission 요청(`allow_once`→`once`). **타임아웃/실패 시 기본 deny**.

---

## 6. 언제 무엇을

```
단순 "프롬프트 만들어 Claude에 던지고 결과 회수"  → CLI 헤드리스(§3) ⭐
멀티턴·피드백 루프(검증 후 수정 위임)            → CLI + --resume/--fork-session
에디터 안에서 Hermes를 코딩 에이전트로            → ACP 서버 모드(hermes acp)
Claude에 우리 도구(사내 API 등)를 쥐여주기        → MCP(--mcp-config)
무인 자동 실행                                    → 권한 모드 + 샌드박스 필수(§3.1)
```

---

## 7. 관련
- [[Autonomous-Agent-Runtimes]] — 모델 vs 런타임 계층, OpenClaw/Hermes 비교
- [[Multi-Agent-Coding-Team]] — 기획/Coder/QA 역할 분담(오케스트레이터=검증자)
- [[Hermes-Agent]] — Hermes function calling·툴 루프
- [[Local-LLM-API-Deployment]] — 로컬 Hermes/Gemma 구동(오케스트레이터 백엔드)
- [[Agent-Security]] — 위임 시 권한·샌드박싱·무인 실행 통제
- [[MCP-Skill]] — MCP 서버/도구 연결
- [[Agent-Harness]] — 에이전트 루프·가드레일 직접 구현
- [[Code-RAG]] — 코딩 에이전트가 쓰는 코드 검색측(AST·심볼·콜그래프)
