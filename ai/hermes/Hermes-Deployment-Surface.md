---
tags:
  - ai
  - agent
  - hermes
  - deployment
  - api
created: 2026-06-19
---

# Hermes 배포 표면 (API 서버 + 멀티플랫폼 게이트웨이)

> [!summary] 한 줄 요약
> **툴 장착 에이전트를 사용자·프론트엔드에 노출하는 "마지막 1마일"** — ① OpenAI 호환 **API 서버**(Open WebUI·LobeChat 등이 Hermes를 *풀 툴셋 백엔드*로), ② **멀티플랫폼 게이트웨이**(Telegram/Discord/Slack/WhatsApp 봇). 개발(CLI)·위임(orchestration)을 넘어 **상시 가동 어시스턴트**가 되는 단계. (`~/.hermes` 기준)

> raw 모델 서빙은 [[Local-LLM-API-Deployment]], 에이전트→에이전트 위임은 [[Orchestrating-Claude-Code]]. 이 노트는 **에이전트→사용자 노출**.

---

## 1. API 서버 — OpenAI 호환 백엔드

```
hermes를 OpenAI 형식 HTTP 엔드포인트로 노출 → 그 포맷을 쓰는 "수백 개" 프론트엔드가 백엔드로 사용
  Open WebUI · LobeChat · LibreChat · NextChat · ChatBox ...
요청은 에이전트의 "풀 툴셋"으로 처리(terminal·file·web·memory·skills) 후 최종 응답
스트리밍 시 tool progress가 inline → 프론트가 "에이전트가 뭐 하는지" 표시
```
```bash
# 임의 OpenAI 호환 클라이언트가 가리킴
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer <key>" -d '{...}'
# CORS·키 설정(API_SERVER_*), /v1/chat/completions 등 엔드포인트
```
> [!important] [[Local-LLM-API-Deployment]]와 구분
> Local-LLM-API는 **raw 모델**(mlx_lm.server 등)을 OpenAI API로 노출. 여기 API 서버는 **에이전트 전체**(툴·메모리·스킬 포함)를 노출한다 — "모델"이 아니라 "에이전트"가 백엔드.

## 2. 멀티플랫폼 게이트웨이 — 봇으로 상주

```
config platform_toolsets: 플랫폼별 전용 툴셋
  discord → hermes-discord / slack → hermes-slack / telegram → hermes-telegram / whatsapp → hermes-whatsapp
각 플랫폼에서 메신저 봇으로 동작 → 채팅으로 에이전트 호출
gateway hooks가 이 표면에서 발화(로깅·알림·웹훅) → [[Agent-Event-Hooks]] §3
```

## 3. 세션 모델

```
group_sessions_per_user: true   — 사용자별 세션 분리(대화·메모리 격리)
max_concurrent_sessions: null   — 동시 세션 상한(기본 무제한)
session_reset: idle/at_hour 기준 리셋
플랫폼·user별로 세션이 갈리고, 각 세션이 영속 메모리를 가짐 → [[Agent-Persistent-Memory]] per-platform
```

## 4. 멀티 프로필 게이트웨이 — 여러 봇 동시 운영

```
profile = 독립된 봇 토큰·세션·메모리·정체성 묶음
한 머신에서 여러 프로필을 managed service(launchd/systemd)로 동시 가동
  → "여러 named 봇"(inbox-triage·ops-review ...) 각자 상주
  → Kanban의 named 프로필 협업과 결합([[Agent-Kanban]])
운영 관심사: 일괄 시작·프로필 통합 로그·sleep 방지·서비스 복구
```

## 5. 연결 — 배포 표면이 묶는 것

```
API 서버    ←→ 프론트엔드(웹 UI)
게이트웨이  ←→ 메신저(상시 봇) + gateway hooks([[Agent-Event-Hooks]])
세션        ←→ per-user/platform 메모리([[Agent-Persistent-Memory]])
cron 결과   → 게이트웨이 타깃으로 전달([[Agent-Scheduled-Execution]])
멀티 프로필 → Kanban 협업([[Agent-Kanban]])
→ 런타임·기억·확장·조율이 "사용자에게 닿는" 지점이 배포 표면이다.
```

## 6. Hermes 전 생애주기에서의 위치

```
개발   : CLI / TUI (로컬 대화·작업)
위임   : 오케스트레이터→Claude Code 등([[Orchestrating-Claude-Code]])
자율   : cron 예약([[Agent-Scheduled-Execution]]) · Kanban([[Agent-Kanban]])
배포   : API 서버(프론트엔드 백엔드) · 게이트웨이(메신저 봇)  ← 이 노트
→ 같은 에이전트(툴·메모리·스킬)를 어느 표면으로 내보내느냐의 차이.
```

---

## 7. 관련
- [[Local-LLM-API-Deployment]] — raw 모델 OpenAI API(여기는 에이전트 전체 노출, 구분)
- [[Agent-Event-Hooks]] — gateway hooks(게이트웨이 표면에서 발화)
- [[Agent-Persistent-Memory]] — per-user/platform 세션 메모리
- [[Agent-Scheduled-Execution]] — cron 결과를 게이트웨이로 전달
- [[Agent-Kanban]] — 멀티 프로필 봇 협업
- [[Orchestrating-Claude-Code]] — 에이전트→에이전트 위임(배포와 다른 축)
- [[vLLM]] — 프로덕션 모델 서빙(API 서버 하단 백엔드 후보)
