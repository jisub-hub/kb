---
tags:
  - ai
  - agent
  - multi-agent
  - coding
  - architecture
created: 2026-06-18
---

# 멀티 에이전트 코딩 팀 구성

> [!summary] 한 줄 요약
> OpenClaw(오케스트레이터)에 역할별 모델(기획·Coder·QA)을 붙여 코드를 자동 작성·검증하는 멀티 에이전트 팀. 구조 자체는 교과서적이지만, **프로덕션으로 가려면 ① 샌드박싱 ② 루프 한도가 필수**다(없으면 자율 코딩은 사고난다). 계층(Hermes는 모델), VRAM 동시구동, QA 객관성, 하이브리드를 더하면 실전급.

---

## 1. 구조 개요

```
OpenClaw (유일한 런타임 / 오케스트레이터, 메시징·UI 창구)
  └ 역할별 "모델" 호출 (페르소나 = 시스템 프롬프트):
      기획/조율  → Hermes 모델     (요구 분석·설계·에이전트 조율)
      코딩       → Qwen2.5-Coder   (파일 작성, file_write)
      QA         → 검증 모델        (테스트 실행, shell_exec + read-only)
```

> 핵심: 단일 에이전트보다 **역할 분담 + 권한 분리 + 피드백 루프**로 안정성↑. [[Agent-Harness]]의 멀티 에이전트 패턴.

---

## 2. ⚠️ 계층 정리 (가장 먼저)

```
OpenClaw + "Hermes Agent 프레임워크" = 런타임 위에 런타임 = 충돌
올바른 구성: OpenClaw가 유일한 런타임, 그 안에서 역할별 "모델"을 호출
  → Hermes는 "조율 역할의 모델"(프레임워크 X)
  → Coder=Qwen Coder 모델, QA=검증 모델
```
→ [[Autonomous-Agent-Runtimes]] 모델 vs 런타임 계층.

---

## 3. 역할 분담

| 에이전트 | 역할 | 권한 |
|----------|------|------|
| **OpenClaw** | 접수·세션·라우팅·워크플로우 제어 | 오케스트레이션 |
| **기획(Hermes)** | 요구 분석, 아키텍처 설계, 조율 | 읽기·계획 |
| **Coder(Qwen)** | 실제 소스 작성·저장 | `file_write`, `file_edit` |
| **QA** | 테스트·린트 실행, 피드백 | `file_read`, `shell_exec` (쓰기 X) |

> 권한 분리 = 최소권한 원칙. Coder는 쓰기, QA는 실행만 — 서로 견제. → [[Agent-Security]]

---

## 4. 워크플로우 (피드백 루프)

```
사용자 요청 (Slack/UI)
  → OpenClaw 접수·세션 생성 → 기획(Hermes) 설계
  → Coder가 워크스페이스에 코드 작성
  → QA가 pytest/린터 shell_exec 실행
     ├─ 실패: 에러 로그 → Coder 수정 (반복)  ← 루프 한도 필수(6절)
     └─ 성공: 결과+리포트 → OpenClaw → 사용자 회신
```

---

## 5. 보안·샌드박싱 (프로덕션 필수) ⭐

```
QA의 shell_exec + Coder의 file_write = 내 머신에서 실제 실행·파일 변경
→ 격리 환경(컨테이너/VM) 안에서만 실행
→ 위험 명령(rm·외부요청·시스템변경)은 승인 게이트 또는 차단
→ 생성 코드를 무통제로 실행하면 사고 (자가개선 코드도 동일 통제)
```
→ [[Agent-Security]] 최소권한·샌드박싱·승인 게이트·감사.

---

## 6. 루프 안전장치 (무한 루프 방지)

```
Coder↔QA "성공할 때까지 반복" → 못 고치면 무한 루프·토큰 폭주
필수:
  - 반복 한도 (예: 5회 실패 시 중단)
  - 토큰·시간 예산
  - N회 실패 → 사람에게 에스컬레이션 (human-in-the-loop)
```
→ [[Agent-Harness]] 루프 한도·가드레일.

---

## 7. QA 자기참조 함정

```
QA가 테스트를 "직접 생성" → 자기가 짠 테스트를 자기가 통과시킴 (무의미)
→ acceptance criteria(통과 기준)는 사람이 정의하거나 요구사항에서 도출
→ QA는 "주어진 기준"으로 검증, 기준 자체를 만들지 않게
→ 가능하면 기준 테스트(골든)는 별도 고정
```

---

## 8. 모델·VRAM 현실 + 하이브리드

```
VRAM: Hermes 70B + Qwen Coder 32B + QA 모델 동시 = 폭발
  현실안: 모델 1~2개를 역할별 시스템 프롬프트로 재사용(페르소나 전환)
          또는 Coder만 강한 모델, 나머지 경량 공유
  → DGX Spark 128GB에 32B+70B 동시는 빠듯 → [[GPU-Inference-Decision-Guide]]

품질: 로컬 32B Coder가 복잡 작업에서 막힘
  → 어려운 작업만 클라우드(Claude/GPT) 폴백 (하이브리드 라우팅)
  → [[Autonomous-Agent-Runtimes]] 6.5절
```

---

## 9. 페르소나 & 권한 구성 (OpenClaw 파일 기반)

```
~/.openclaw/workspace/
  CODER.md  "너는 클린코드 지향 시니어 백엔드. 코드는 지정 폴더에 저장."
  QA.md     "너는 깐깐한 QA. 예외처리 확인, 쉘로 테스트 직접 실행. 코드 수정 금지."

도구 권한:
  Coder → file_write, file_edit (쓰기)
  QA    → file_read, shell_exec (실행, 쓰기 박탈)
  기획  → 읽기·계획 (실행 권한 없음)
```

---

## 10. 보정된 구조 (실전급)

```
OpenClaw (유일 런타임, 메시징 UI)
  ├ 페르소나별 모델 호출 (기획=Hermes / Coder=Qwen / QA=검증)
  ├ 격리 실행 (컨테이너) + 위험 명령 승인 게이트       ← ③ 보안
  ├ 루프 한도 + 토큰 예산 + 실패 시 에스컬레이션        ← ④ 안전
  ├ acceptance criteria는 사람/요구사항에서             ← ⑤ 객관성
  └ 어려운 작업 클라우드 폴백(선택)                      ← ⑥ 품질
```

---

## 11. 단계별 도입

```
토이/개인용:   역할 분담 + 권한 분리 + ③샌드박싱 + ④루프 한도 (이거면 충분)
팀/프로덕션:   + ⑤QA 객관성 + ⑥하이브리드 + 감사 로그 + 멀티테넌트
              → [[Multi-Tenant-Memory-Agent]]
```

> 시작은 가볍게(③④만), 키울 때 ⑤⑥ 추가. 구조는 좋으니 **안전장치만 갖추면 강력한 코딩 아웃소싱 팀**이 된다.

---

## 12. 관련
- [[Autonomous-Agent-Runtimes]] — OpenClaw 런타임, 모델 vs 런타임 계층
- [[Agent-Security]] — 샌드박싱·승인 게이트·도구 권한 (③ 필수)
- [[Agent-Harness]] — 에이전트 루프·루프 한도·가드레일 (④)
- [[Hermes-Agent]] — function calling (조율·도구 호출)
- [[GPU-Inference-Decision-Guide]] · [[Local-LLM-API-Deployment]] — 로컬 모델 구동
- [[Multi-Tenant-Memory-Agent]] — 팀 확장 시 격리·메모리
- [[Tracing]] — 에이전트 멀티홉(기획→Coder→QA) 추적·디버깅 (§6.3)
- [[Agent-Kanban]] — durable 보드로 이 팀 파이프라인을 구현(재개·역할 핸드오프)
