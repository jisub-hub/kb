---
tags:
  - ai
  - agent
  - hermes
  - multi-agent
  - orchestration
created: 2026-06-19
---

# Kanban — durable 멀티 에이전트 조율 보드

> [!summary] 한 줄 요약
> 여러 **named 프로필**(독립 OS 프로세스)이 **영속 SQLite 보드**(`~/.hermes/kanban.db`)로 협업하는 프리미티브. `delegate_task`(RPC fork→join, 휘발)가 못 하는 **재개·휴먼인루프·영속 정체성·감사**를 담당한다. 작업=row, 핸드오프=row → 누구나(다른 프로필·사람) 읽고 쓴다. (`~/.hermes` 기준)

> in-process 서브에이전트는 [[Hermes-Runtime-Internals]] delegate_task, 예약 실행은 [[Agent-Scheduled-Execution]], 개념적 역할 분담은 [[Multi-Agent-Coding-Team]]. 이 노트는 **"경계를 넘는·살아남는" 조율 축**.

---

## 1. delegate_task vs Kanban — 다른 프리미티브 ⭐

| | `delegate_task` | **Kanban** |
|---|---|---|
| 형태 | RPC 호출(fork→join) | **durable 큐 + 상태머신** |
| 부모 | 자식 반환까지 블록 | `create` 후 fire-and-forget |
| 자식 정체성 | 익명 서브에이전트 | **named 프로필**(영속 메모리) |
| 재개성 | 없음(실패=실패) | block→unblock→재실행, **crash→reclaim** |
| 휴먼인루프 | ❌ | comment/unblock 언제든 |
| 작업당 에이전트 | 1호출=1서브에이전트 | 수명 동안 N개(재시도·리뷰·후속) |
| 감사추적 | 컨텍스트 압축 시 소실 | **SQLite row 영구** |
| 조율 | 계층적(caller→callee) | **peer**(누구나 어떤 작업이든 R/W) |

```
한 줄 구분: delegate_task = 함수 호출 / Kanban = 작업 큐(모든 핸드오프가 row)
delegate_task 쓸 때: 부모가 짧은 추론 답을 받아 이어가야 할 때, 사람 없이, 결과가 부모 컨텍스트로
Kanban 쓸 때      : 작업이 에이전트 경계를 넘고·재시작을 견디고·사람 개입 가능·다른 역할이 이어받고·사후 추적 필요
공존: kanban worker가 실행 중 내부적으로 delegate_task를 부를 수 있음
```

## 2. 핵심 개념

```
Task : row — title·body·assignee(프로필명 1개)·status·tenant(선택)·idempotency key(재시도 dedup)
  status: triage → todo → ready → running → blocked → done → archived
Link : task_links row — parent→child 의존. 모든 parent가 done이면 dispatcher가 todo→ready 승격
Workspace: scratch(기본, 완료 시 삭제·휘발) / worktree: / dir:<path>(보존)
```

## 3. 두 표면 — 모델은 툴, 사람은 CLI

```
모델: kanban_* 툴셋으로 직접 — kanban_show/list/create/complete/block/unblock/heartbeat/comment/link
      (dispatcher가 worker를 이 툴들과 함께 스폰. shell로 `hermes kanban` 호출 ❌)
사람·cron·대시보드: `hermes kanban …` / `/kanban` / 웹 대시보드
→ 둘 다 같은 kanban_db 레이어 → 읽기 일관·쓰기 드리프트 없음
```

## 4. Dispatcher — 조율 엔진

```
게이트웨이 안에서 도는 long-lived 루프(기본 60초마다):
  ① stale 클레임 회수  ② crash된 worker 회수(PID 사라졌는데 TTL 남음)
  ③ ready 작업 승격     ④ atomic claim 후 assigned 프로필 스폰
worker는 HERMES_KANBAN_BOARD 고정 스폰 → 다른 보드 못 봄(격리)
⚠️ 같은 작업에 연속 스폰 실패 kanban.failure_limit(기본 2) 회 → 자동 block(마지막 에러를 사유로)
   → 존재하지 않는 프로필·마운트 불가 워크스페이스로 thrashing 방지
```

## 5. 생존성 — heartbeat·reclaim ⚠️

```
긴 작업은 kanban_heartbeat(note=...)로 생존 신호
  1시간 넘는 작업 → 최소 매시간 heartbeat 호출
  dispatch_stale_timeout(기본 4h) 지나고 최근 1h heartbeat 없으면 → crash로 간주, reclaim
reclaim은 benign: 작업이 ready로 돌아가 재배치(실패 카운터 안 올림) — 단 현재 run 진행분은 잃음
```
> crash 복원력이 핵심 — 프로세스가 죽어도 작업은 보드에 남아 다른 worker가 이어받는다(delegate_task엔 없는 것).

## 6. 워크로드 — 언제 Kanban인가

```
- 리서치 트리아지: 병렬 리서처 + 분석가 + 작성자, human-in-loop
- 스케줄 ops: 매일 브리프가 몇 주에 걸쳐 저널 축적 (+[[Agent-Scheduled-Execution]] cron)
- digital twin: inbox-triage·ops-review 같은 영속 named 비서(시간 따라 메모리 축적)
- 엔지니어링 파이프라인: 분해 → 병렬 worktree 구현 → 리뷰 → 반복 → PR
- fleet work: 한 전문가가 N 대상 관리(소셜 50계정·서비스 12개)
```

## 7. 세 가지 조율 프리미티브 정리

```
delegate_task : in-process·동기·휘발     — 짧은 하위추론, 부모가 결과 필요    ([[Hermes-Runtime-Internals]])
cron          : 시간 기반 자율 트리거    — 예약·반복·무인                    ([[Agent-Scheduled-Execution]])
Kanban        : durable·peer·재개 가능   — 경계 넘는·살아남는·사람 개입 협업
→ "주기"가 아니라 "수명·정체성·재개성"이 선택 기준.
```

---

## 8. 관련
- [[Hermes-Runtime-Internals]] — delegate_task(in-process 서브에이전트, 대비 프리미티브)
- [[Agent-Scheduled-Execution]] — cron 예약(Kanban 작업 생성 트리거로 결합)
- [[Multi-Agent-Coding-Team]] — 역할별 코딩 팀(Kanban로 구현 시 파이프라인)
- [[Agent-Persistent-Memory]] — named 프로필의 영속 메모리(digital twin)
- [[Agent-Security]] — 다중 프로필·자율 실행의 권한·격리
- [[Autonomous-Agent-Runtimes]] — 자율 런타임 일반
