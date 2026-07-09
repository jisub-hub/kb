---
tags:
  - ai
  - agent
  - openclaw
  - autonomous
  - runtime
  - overview
created: 2026-06-24
---

# OpenClaw 개요 — 자율 에이전트 런타임

> [!summary] 한 줄 요약
> **OpenClaw는 LLM을 갈아끼워 내 머신에서 실제 액션(웹·파일·셸)을 수행하는 자율 에이전트 "런타임(몸체)"**이다. 텍스트 생성이 아니라 행동이 핵심이고, 인터페이스가 메시징 앱이라 "개인 비서" 성격이 강하다. LLM은 외부 연결(Claude/GPT/DeepSeek/로컬)이므로, **모델(두뇌)과 런타임(몸체)을 먼저 구분**해야 한다. Hermes와의 전체 비교·출처는 [[Autonomous-Agent-Runtimes]].

---

## 1. 정체 · 출처

```
개발: Peter Steinberger (오스트리아), 2025.11 공개, 라이선스 MIT
규모: 2026.01 기준 100,000+ GitHub stars (출시 첫 주 급상승)
정체: 자율 에이전트 "런타임" — LLM은 외부 연결(Claude/GPT/DeepSeek/로컬)
```
> 핵심 차별: **텍스트 생성이 아니라 내 머신에서 실제 액션을 수행.** 메시징 앱이 인터페이스 → 개인 비서.

---

## 2. 아키텍처 — 모델(두뇌) vs 런타임(몸체)

```
런타임 (몸체) = OpenClaw      ← 액션 실행·메모리·자가개선
   ├ LLM 호출 (두뇌를 갈아끼움)
   ├ 도구 실행 (웹·파일·셸)
   ├ 메모리 (장기 기억)
   └ 자가개선 (필요 스킬을 스스로 코드로 작성)
        │ 외부 LLM 연결
        ▼
모델 (두뇌) = Hermes / Claude / GPT / Qwen / DeepSeek
```
> "Hermes **모델**로 OpenClaw 대체?" → ❌ 범주 오류(두뇌 vs 몸체). 올바른 관계는 **OpenClaw 런타임의 백엔드 LLM으로 모델을 꽂는 것**.

---

## 3. 인터페이스 · 액션 · 특징

| 축 | 내용 |
|---|---|
| **UI** | 메시징 플랫폼 — Signal / Telegram / Discord / WhatsApp |
| **액션** | 웹 브라우징·폼 작성·데이터 추출 / 파일 읽기·쓰기 / 셸·스크립트 실행 |
| **자가개선** | 필요한 스킬을 스스로 코드로 작성 |
| **메모리** | 장기 기억(사용자 선호 누적) |
| **LLM** | 외부 연결(Claude/GPT/DeepSeek/로컬), provider 교체 가능 |
| **라이선스** | MIT |

---

## 4. 백엔드 모델 선택 & 로컬 한계

자율 런타임은 챗봇보다 **모델에 까다롭다** — 멀티스텝 + 도구 호출 + 긴 컨텍스트 + (자가개선)을 요구한다.

```
에이전트가 모델에 요구: ① function calling 신뢰성 ② 멀티스텝 추론
                        ③ 긴 컨텍스트 ④ 지시 준수(드리프트 없음)
→ 작은 모델은 ②④에서 무너짐(도구 오호출·루프) → 멀티스텝일수록 실수 누적·증폭
```

| 등급 | 모델 | 자율 작업 적합도 |
|------|------|----------------|
| 최상(품질) | Claude Sonnet/Opus, GPT-5.x | 복잡 자율 안정 — API 비용·외부 전송 |
| **로컬 실용 하한** | Qwen2.5 72B, Llama 3.3 70B, Hermes 3 70B | 도구·멀티스텝 쓸만 (M Ultra/96GB급) |
| 로컬 중급 | Qwen2.5 32B, Hermes 4.3 36B | 단순~중간 자율 (M Max급) |
| 로컬 최소 | 8B~14B | 단일 도구 호출만 — 복잡 자율은 무리 |

> **OpenClaw "잘" 쓰려면 70B급이 로컬 실용 하한.** 단순~중간 자율은 로컬로 충분하나, 복잡 추론은 로컬이 폐쇄 모델에 밀린다.

### 하이브리드 라우팅 (현실적 정답)
```
"전부 로컬" or "전부 클라우드"가 아니라 작업 난이도로 분기:
  단순·반복·프라이버시 → 로컬(Qwen2.5 32~72B / Hermes 70B)
  복잡·고난도 추론     → 클라우드(Claude/GPT) 폴백
  → OpenClaw provider를 작업별로 라우팅(게이트웨이 패턴)
```
→ 모델 라우팅 [[LLMOps]], 로컬 하드웨어 [[GPU-Inference-Decision-Guide]] · [[Apple-Silicon-Inference]].

---

## 5. 사용 예시

### 5.1 OpenClaw + 로컬 Hermes/Qwen (비용 0·프라이버시 개인 비서)
```bash
# 1. 로컬 모델 서빙 (OpenAI 호환)
ollama pull hermes3:8b && ollama serve          # :11434
# 2. OpenClaw: LLM provider = local OpenAI-compatible(http://localhost:11434/v1)
#    messaging = telegram(봇 토큰)
```
```
[Telegram] "내일 9시 회의자료 정리해 메일초안 만들어줘"
  → OpenClaw가 LLM에 질의 → 도구 호출 계획 → 파일 읽기·요약·메일초안(파일 쓰기)
  → Telegram 회신   ※ 데이터 로컬 유지(프라이버시), API 비용 0
```
백엔드 서빙은 [[Local-LLM-API-Deployment]].

### 5.2 OpenClaw + 클라우드 모델 (품질 우선)
```
복잡 멀티스텝(리팩터링·리서치)은 추론 깊이 중요 → 백엔드 LLM을 Claude/GPT로
단, 데이터 외부 전송(프라이버시 trade-off) + API 비용
```

### 5.3 코딩 팀 오케스트레이터
OpenClaw에 역할별 모델(기획·Coder·QA)을 붙인 멀티 에이전트 코딩 팀 → [[Multi-Agent-Coding-Team]].

---

## 6. 보안 주의 ⚠️

> [!warning] 자율 런타임의 위험
> OpenClaw류는 **셸·파일·웹에 실제 액션**을 한다 → 권한·샌드박싱·승인 게이트 필수. 자가개선(스스로 코드 작성)은 강력하지만 통제되지 않으면 위험. **최소 권한·격리 환경**에서 운용. → [[Agent-Security]] · [[Security-Checklist]]

---

## 7. vs Hermes · 직접 구현

```
OpenClaw vs Hermes Agent = 레이어가 겹치는 두 런타임(경쟁 아님)
  메시징 비서·액션 생태계 성숙도 → OpenClaw 우위
  NousResearch 모델 긴밀 통합   → Hermes Agent 우위
통제된 커스텀 에이전트면 런타임 없이 직접 구현도 선택 → [[Agent-Harness]]
```
- 전체 비교표·대체가능성 판정·출처: [[Autonomous-Agent-Runtimes]]
- Hermes 프레임워크 전반: [[ai/hermes/_index|Hermes 에이전트 MOC]]

---

## 8. 관련
- [[Autonomous-Agent-Runtimes]] — OpenClaw vs Hermes 전체 비교(모델 vs 런타임)·출처 ⭐
- [[Multi-Agent-Coding-Team]] — OpenClaw 오케스트레이터 + 역할별 모델 코딩 팀
- [[Local-LLM-API-Deployment]] — 로컬 모델을 OpenAI 호환 API로(런타임 백엔드)
- [[Open-Source-Models]] — 백엔드 모델 선택(Hermes/Qwen/Llama)
- [[Agent-Security]] — 자율 런타임 권한·샌드박싱·자가개선 통제
- [[ai/hermes/_index|Hermes 에이전트 MOC]] — 자매 런타임/프레임워크
