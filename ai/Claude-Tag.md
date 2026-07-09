---
tags:
  - ai
  - claude
  - anthropic
  - agent
  - slack
  - deployment
  - collaboration
created: 2026-06-24
---

# Claude Tag — Slack 안의 AI 팀원 (분석)

> [!summary] 한 줄 요약
> **Claude Tag**(Anthropic, 2026-06-23 발표)는 Claude를 **Slack 채널의 팀원**으로 상주시키는 기능이다. `@Claude`로 태그해 작업을 위임하면, Claude가 **채널 맥락을 기억하며** 일을 처리한다. 1:1 비서가 아니라 **채널 단위 "멀티플레이어" 공유 에이전트** — 누구나 진행 상황을 보고 이어받을 수 있다. Claude Enterprise·Team 대상 Slack 베타.

---

## 1. 무엇인가 (사실)
- **위임 UX**: 채널에서 `@Claude` 태그 → 인사이트 제공·작업 할당. 사람이 다른 일 하는 동안 Claude가 처리.
- **채널 메모리**: 채널을 따라다니며 관련 정보를 기억 → 매번 처음부터 설명할 필요 없음. 미래 작업도 계획.
- **멀티플레이어**: 한 채널에 **하나의 Claude**가 모두와 상호작용 → 누구나 진행 중인 작업을 보고, 끊긴 지점부터 이어받음.
- **엔터프라이즈 제어**: 관리자가 채널별로 **접근 가능한 도구·정보를 지정**, 용도별 **별도 Claude 정체성** 생성.
- **가용성**: Claude Enterprise·Team 고객 대상 **Slack 베타**.
- **내부 도입**: Anthropic 제품팀 코드의 **65%**를 내부판 Claude Tag가 작성.

---

## 2. 분석 — 무엇이 새로운가
1. **1:1 어시스턴트 → 채널 공유 에이전트.** 기존 챗봇/코파일럿은 개인-에이전트 1:1이었다. Claude Tag는 **팀 공유 컨텍스트**(채널 = 메모리 경계)를 가진 "동료" 모델로, 협업의 단위가 사람-AI가 아니라 **채널-AI**가 된다.
2. **메신저가 곧 배포 표면.** "에이전트를 메신저 봇으로 상주"는 OpenClaw·Hermes 게이트웨이가 이미 추구한 패턴([[Hermes-Deployment-Surface]]). Claude Tag는 그것의 **Anthropic 공식·관리형(hosted)** 버전 — 셀프호스팅 게이트웨이와 정확히 대비된다.
3. **컨텍스트 누적이 핵심 가치.** 채널 로그를 장기 메모리로 삼아 "회사를 학습"한다(테크크런치 표제). 이는 [[Agent-Persistent-Memory]]·[[Multi-Tenant-Memory-Agent]]의 팀 KB 누적·테넌트 격리 문제와 동일 선상 — 단 Anthropic이 호스팅.
4. **관리자 제어 = 거버넌스 내장.** 채널별 도구/정보 스코프, 별도 정체성 → [[Agent-Security]]의 최소권한·격리를 제품 기능으로 흡수.

---

## 3. 자가호스팅(Hermes 게이트웨이) vs Claude Tag (관리형)
| 축 | Hermes 게이트웨이 봇 | Claude Tag |
|---|---|---|
| 운영 | 자가호스팅(`~/.hermes`, launchd/systemd) | Anthropic 관리형(Slack 베타) |
| 모델 | provider 자유(로컬/클라우드) | Claude 고정 |
| 데이터 | 로컬 통제 가능(프라이버시) | Anthropic·Slack 경유 |
| 멀티플랫폼 | Telegram/Discord/Slack/WhatsApp | Slack(현재) |
| 커스텀 | 툴·스킬·플러그인 완전 통제 | 관리자 스코프 설정 수준 |
| 비용 | 인프라+모델(로컬 0 가능) | Enterprise/Team 구독 |
| 메모리 | 직접 설계(MEMORY/RLS) | 채널 메모리 내장 |
| 강점 | 통제·프라이버시·멀티모델 | 설치 0·팀 공유·거버넌스 내장 |

> 정리: **통제·프라이버시·멀티모델이면 자가호스팅 게이트웨이, 설치 부담 없이 팀 협업·거버넌스면 Claude Tag.** 둘 다 "에이전트를 채팅 표면에 상주"라는 같은 패턴의 다른 운영 모델.

---

## 4. 시사점 (이 볼트 맥락)
- 우리는 이미 `claude -p` 구독 재사용([[Claude-Code-Headless]])·Hermes 게이트웨이([[Hermes-Deployment-Surface]])로 "에이전트를 노출"하는 길을 본다. Claude Tag는 그 **관리형 대안**이라, **build(자가호스팅) vs buy(Claude Tag)** 선택지로 비교 가능.
- 멀티플레이어 채널 메모리는 [[Multi-Tenant-Memory-Agent]]의 팀 KB·memory poisoning 논점을 그대로 안고 있다 — 관리형이라도 채널 컨텍스트 오염·권한 누출 검토 필요.
- 코드 65% 내부작성 수치는 [[Harness-Engineering]]의 "하네스(환경·피드백)가 갖춰지면 에이전트 실용성이 급등"을 뒷받침하는 사례.

---

## 5. 관련
- [[Hermes-Deployment-Surface]] — 자가호스팅 메신저 게이트웨이(자가호스팅 대응물)
- [[Agent-Persistent-Memory]] · [[Multi-Tenant-Memory-Agent]] — 채널/팀 메모리·격리·poisoning
- [[Claude-Code-Headless]] — `claude -p` 구독 재사용(또 다른 Claude 활용 경로)
- [[Agent-Security]] — 도구/정보 스코프·최소권한
- [[Harness-Engineering]] — 하네스가 에이전트 실용성을 가른다

---

## 출처
- [Introducing Claude Tag — Anthropic](https://www.anthropic.com/news/introducing-claude-tag)
- [Anthropic's Claude Tag is learning your company, one Slack message at a time — TechCrunch (2026.06.23)](https://techcrunch.com/2026/06/23/anthropics-claude-tag-is-learning-your-company-one-slack-message-at-a-time/)
- [Anthropic launches Claude Tag, a tool that works like a virtual employee within Slack — Fortune (2026.06.23)](https://fortune.com/2026/06/23/anthropic-claude-tag-virtual-employee-tool-slack/)
- [Anthropic introduces Claude Tag, a new AI teammate for Slack — Neowin](https://www.neowin.net/news/anthropic-introduces-claude-tag-a-new-ai-teammate-for-slack/)
