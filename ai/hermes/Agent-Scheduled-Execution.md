---
tags:
  - ai
  - agent
  - hermes
  - cron
  - automation
created: 2026-06-19
---

# 에이전트 예약 자율 실행 (Cron)

> [!summary] 한 줄 요약
> 에이전트를 **사람 없이 일정에 따라** 돌리는 메커니즘 — 자연어/cron식으로 작업을 예약하고, 스킬을 붙이고, 결과를 채팅·파일로 받는다. 핵심 설계는 두 가지: **no-agent 모드**(LLM 없이 스크립트만)와 **runaway 방지**(예약 실행이 또 예약을 못 만들게 차단). Hermes(`~/.hermes`) 기준.

> 무인 실행 통제는 [[Agent-Security]], 루프 가드레일은 [[Hermes-Runtime-Internals]], 직접 위임 패턴은 [[Orchestrating-Claude-Code]]. 이 노트는 **"스스로 깨어나 일하는"** 자율 운영면.

---

## 1. 단일 `cronjob` 툴 — 자연어로 관리

```
하나의 cronjob 툴이 action 방식으로 전부 처리(별도 schedule/list/remove 툴 없음):
  add · pause · resume · edit · trigger · remove
→ "매시간 피드 요약해줘"처럼 평문으로 요청하면 에이전트가 cronjob 툴로 등록
```
```bash
/cron add 30m "빌드 확인 리마인드"
/cron add "every 2h" "서버 상태 체크"
/cron add "every 1h" "새 피드 요약" --skill blogwatcher
hermes cron create "every 1h" "..." --skill blogwatcher --skill maps --name "Skill combo"
```

## 2. 스케줄·스킬·결과 전달

```
스케줄: one-shot(30m) / 반복(every 2h) / cron식 / 자연어
스킬 첨부: 작업당 0~다수 skill 부착 → 예약 작업이 특정 절차지식을 들고 실행
결과 전달: 원 채팅 / 로컬 파일 / 설정된 플랫폼 타깃(Telegram·Slack 등)
실행 형태: 기본은 "fresh agent 세션"(정적 툴 목록으로 새로 시작)
```

## 3. no-agent 모드 — LLM 없이 스크립트만 ⭐

```
no-agent 모드: 스케줄에 "스크립트"만 걸고 LLM은 전혀 안 부름
  → stdout을 그대로 전달. 비용 0(추론 없음)·결정론적
언제: 단순 체크·수집·알림처럼 추론이 불필요한 정기 작업
  (예: 디스크 사용량 보고, 헬스체크) → 굳이 LLM 토큰 태우지 말 것
대비: fresh-session 모드는 판단·도구사용이 필요한 작업에만
```
> 비용 원칙: **추론이 필요 없으면 LLM을 호출하지 마라.** 정기 작업의 상당수는 스크립트로 충분 → no-agent로.

## 4. ⚠️ Runaway 방지 — 예약이 예약을 못 낳는다

> [!danger] cron 실행 세션 안에서는 **cron 관리 툴이 비활성화**된다
> 예약 실행이 또 다른 cron을 만들면 → 기하급수적 폭주(self-replicating jobs). Hermes는 cron 실행 컨텍스트에서 cronjob 생성/관리 툴을 꺼서 **재귀 스케줄링을 원천 차단**한다.

```
무인 자율의 제1 위험 = 통제 없는 자기증식
  방어1: 예약 세션 → 예약 생성 불가(재귀 차단)
  방어2: 루프 가드레일([[Hermes-Runtime-Internals]] §3) — 무진전 반복 차단
  방어3: 권한·샌드박스([[Agent-Security]]) — 무인 실행의 행동 범위 제한
```

## 5. 무인 실행 인증

```
cron은 `hermes model`이 선택한 provider를 그대로 사용
  무인 환경 권장: OAuth 자동 갱신되는 방식(포털 등) → 토큰 만료로 한밤중 작업 실패 방지
```

## 6. 내 반자율 루프와의 대비 (메타)

```
이 vault의 5분 강화 루프 vs Hermes cron — 같은 "예약 자율 실행"의 두 구현:
  공통: 일정 주기로 깨어나 작업 → 결과 기록
  차이(이 루프의 안전장치):
    - 신규 지식 작성은 사람 승인 게이트(REVIEW-QUEUE [x])만 — 반자율
    - 위생/조정만 자동(소규모) → 품질 표류 방지
  cron의 안전장치: 재귀 cron 차단 + 가드레일 + 권한 → 폭주 방지
→ 자율 운영의 핵심은 "주기"가 아니라 "무엇을 사람이 승인하고 무엇을 막는가"의 설계.
```

---

## 7. 관련
- [[Agent-Security]] — 무인 실행 권한·샌드박싱·승인 게이트
- [[Hermes-Runtime-Internals]] — tool-loop 가드레일(폭주 방지의 다른 축)
- [[Agent-Event-Hooks]] — 라이프사이클 이벤트(예약 실행 관측·알림)
- [[Orchestrating-Claude-Code]] — 위임 실행(예약과 결합 가능)
- [[Hermes-Skills]] — cron 작업에 첨부하는 skill(`--skill`)의 로딩 메커니즘
- [[Agent-Kanban]] — 예약이 durable 보드 작업을 생성·트리거(cron+kanban 결합)
- [[Autonomous-Agent-Runtimes]] — 자율 에이전트 런타임 일반
