---
tags:
  - ai
  - agent
  - autonomous
  - openclaw
  - hermes
  - runtime
created: 2026-06-18
---

# 자율 에이전트 런타임 — OpenClaw vs Hermes Agent (모델 vs 런타임)

> [!summary] 한 줄 요약
> **"모델(두뇌)"과 "런타임(몸체)"은 다른 계층**이다. OpenClaw·Hermes Agent는 LLM을 갈아끼워 실제 액션을 수행하는 *런타임*이고, Hermes 모델은 그 안에 들어가는 *LLM*이다. "Hermes 모델로 OpenClaw 대체"는 범주 오류 — 오히려 **OpenClaw + 로컬 Hermes**가 좋은 조합이다.

---

## 1. 계층 구분 (가장 중요)

```
┌─────────────────────────────────────────────┐
│  런타임 (몸체)  — OpenClaw / Hermes Agent      │  ← 액션 실행·메모리·자가개선
│   ├ LLM 호출 (두뇌를 갈아끼움)                  │
│   ├ 도구 실행 (웹·파일·셸)                      │
│   ├ 메모리 (장기 기억)                          │
│   └ 자가개선 (스킬 자동 생성)                    │
└─────────────────────────────────────────────┘
              │ 외부 LLM 연결
              ▼
┌─────────────────────────────────────────────┐
│  모델 (두뇌)  — Hermes / Claude / GPT / Qwen   │  ← 추론·도구호출 판단
└─────────────────────────────────────────────┘
```

> "Hermes **모델**로 OpenClaw 대체?" → ❌ 범주 오류 (두뇌 vs 몸체).
> "Hermes **Agent 프레임워크**로 OpenClaw 대체?" → △ 같은 런타임 카테고리, 부분 가능.

---

## 2. OpenClaw

```
개발: Peter Steinberger (오스트리아), 2025.11 공개, MIT
규모: 2026.01 100,000+ GitHub stars (출시 첫 주)
정체: 자율 에이전트 "런타임" — LLM은 외부 연결(Claude/GPT/DeepSeek)
UI:  메시징 플랫폼 (Signal / Telegram / Discord / WhatsApp)
액션: 웹 브라우징·폼 작성·데이터 추출 / 파일 읽기·쓰기 / 셸·스크립트 실행
특징: 자가개선(필요 스킬을 스스로 코드로 작성) + 장기 메모리(사용자 선호 기억)
```

핵심 차별: **텍스트 생성이 아니라 내 머신에서 실제 액션을 수행**. 메시징 앱이 인터페이스라 "개인 비서" 성격이 강하다.

---

## 3. Hermes Agent (프레임워크)

```
개발: NousResearch, 2026.02, MIT
규모: 46,000+ GitHub stars
정체: 자율 에이전트 프레임워크 — provider 자유(백엔드 LLM 선택)
특징: 자가개선(self-improving) + 지속 메모리(persistent), "성장하는 에이전트"
```

→ Hermes "모델"(function calling LLM)과 이름만 같고 계층이 다르다. 상세는 [[Hermes-Agent]].

---

## 4. 비교표

| 항목 | OpenClaw | Hermes Agent |
|------|----------|--------------|
| 카테고리 | 자율 에이전트 런타임 | 자율 에이전트 프레임워크 |
| LLM | 외부 연결(Claude/GPT/DeepSeek/로컬) | provider 자유 |
| 대표 UI | **메시징(Signal/TG/Discord/WA)** | CLI/API 중심(추정) |
| 자가개선 | ✅ 스킬 자동 작성 | ✅ self-improving |
| 메모리 | ✅ 장기(선호 기억) | ✅ persistent |
| 액션 실행 | ✅ 웹·파일·셸 (성숙) | 도구 호출 |
| 라이선스 | MIT | MIT |
| 규모 | 100k+ stars (2026.01) | 46k+ stars (2026.02) |
| 강점 | 메시징 비서·액션 생태계 | NousResearch 생태계 |

---

## 5. 대체 가능성 판정

```
① Hermes 모델 → OpenClaw 대체:  ❌ 범주 오류 (모델 ≠ 런타임)
   올바른 관계: OpenClaw 런타임의 백엔드 LLM으로 Hermes 사용 (보완)

② Hermes Agent → OpenClaw 대체:  △ 같은 카테고리, 부분 가능
   - 메시징 비서·액션 실행 성숙도: OpenClaw 우위
   - NousResearch 모델 긴밀 통합: Hermes Agent 우위
```

> 둘은 경쟁이 아니라 **레이어가 겹치는 두 런타임**이고, 모델(Hermes/Qwen/Claude)은 그 아래 갈아끼우는 부품이다.

---

## 6. 사용 예시

### 6.1 OpenClaw + 로컬 Hermes (비용 0·프라이버시 개인 비서)

```
구성:
  Mac: Ollama로 Hermes 3 8B 서빙 (OpenAI 호환 API :11434)  → [[Local-LLM-API-Deployment]]
  OpenClaw: 백엔드 LLM을 로컬 Ollama로 지정
  UI: Telegram 봇

흐름:
  [Telegram] "내일 9시 회의 자료 정리해서 메일 초안 만들어줘"
    → OpenClaw가 LLM(Hermes)에 질의 → 도구 호출 계획 수립
    → 파일 읽기(회의 자료) → 요약 → 메일 초안 작성(파일 쓰기)
    → Telegram으로 결과 회신
  ※ 데이터가 로컬을 안 떠남(프라이버시), API 비용 0
```

```bash
# 1. 로컬 모델 서빙
ollama pull hermes3:8b && ollama serve         # :11434, OpenAI 호환

# 2. OpenClaw 설정 (개념 — 실제 키는 docs 참조)
#    LLM provider = local OpenAI-compatible (http://localhost:11434/v1)
#    messaging = telegram (봇 토큰)
#    → openclaw가 도구(웹/파일/셸)를 실행하며 자율 처리
```

### 6.2 OpenClaw + 클라우드 모델 (품질 우선)

```
복잡한 멀티스텝(코드 리팩터링·리서치)은 추론 깊이가 중요
  → 백엔드 LLM을 Claude/GPT로 지정 (Hermes보다 추론 우위)
  → 단 데이터가 외부로 나감(프라이버시 trade-off) + API 비용
```

### 6.3 Hermes Agent (NousResearch 스택)

```
NousResearch 모델·생태계에 묶인 자율 에이전트가 목적일 때
  → Hermes Agent 프레임워크 + Hermes 모델
  → 지속 메모리로 사용자 맥락 누적, 자가개선으로 스킬 확장
```

### 6.4 직접 구현 (프레임워크 없이)

```
단순·통제된 에이전트면 런타임 프레임워크 없이 직접:
  Hermes 모델(function calling) + 자체 에이전트 루프  → [[Agent-Harness]] / [[Hermes-Agent]]
  → 도구·가드레일·종료조건을 코드로 완전 통제
```

---

## 6.5 백엔드 모델 선택 & 로컬 한계

자율 런타임(OpenClaw 등)은 챗봇보다 **모델에 까다롭다** — 멀티스텝 + 도구 호출 + 긴 컨텍스트 + (자가개선)이 요구된다.

```
에이전트가 모델에 요구:
  ① function calling 신뢰성 (도구 호출 JSON 안 깨짐)
  ② 멀티스텝 추론 (계획→실행→평가→다음)
  ③ 긴 컨텍스트 (도구 결과·기억 누적)
  ④ 지시 준수 (드리프트 없이 역할 유지)
→ 작은 모델은 ②④에서 무너짐 (도구 오호출·루프) → 멀티스텝일수록 실수 누적·증폭
```

### 모델 등급

| 등급 | 모델 | 자율 작업 적합도 |
|------|------|----------------|
| 최상(품질) | Claude Sonnet/Opus, GPT-5.x | 복잡 자율 안정 — 단 API 비용·외부 전송 |
| **로컬 실용 하한** | Qwen2.5 72B, Llama 3.3 70B, Hermes 3 70B | 도구 호출·멀티스텝 쓸만 (M Ultra/96GB급 필요) |
| 로컬 중급 | Qwen2.5 32B, Hermes 4.3 36B | 단순~중간 자율 (M Max급) |
| 로컬 최소 | 8B~14B | 단순 단일 도구 호출만 — 복잡 자율은 무리 |

### "로컬 LLM으로 많은 걸"은 절반만 가능

```
✅ 로컬로 충분: 단순~중간 자율(파일 정리·요약·정해진 워크플로우),
   프라이버시 민감, 비용 0 반복 → Qwen2.5 32~72B / Hermes 70B
⚠️ 로컬이 버거움: 복잡 멀티스텝·모호한 목표·자가개선
   → 로컬 70B도 Claude/GPT보다 명확히 약함 (실패율↑)
❌ 무리: 8B급으로 복잡 자율 → 도구 오호출·루프·환각
```

### 현실적 정답 — 하이브리드 라우팅

```
"전부 로컬" or "전부 클라우드"가 아니라 작업 난이도로 분기:
  단순·반복·프라이버시 → 로컬 (Qwen2.5 32~72B / Hermes 70B)
  복잡·고난도 추론    → 클라우드 (Claude/GPT) 폴백
  → OpenClaw provider를 작업별로 라우팅 (게이트웨이 패턴)
```

→ 모델 라우팅 [[LLMOps]], 로컬 구동 하드웨어 [[GPU-Inference-Decision-Guide]] · [[Apple-Silicon-Inference]].

> 요약: **OpenClaw "잘" 쓰려면 70B급이 로컬 실용 하한.** 단순~중간 자율은 로컬로 충분하나, 복잡 추론은 로컬이 폐쇄 모델에 밀린다 → 하이브리드가 정답.

---

## 7. 선택 가이드

| 목적 | 선택 |
|------|------|
| 메시징 기반 개인 비서(로컬·무료) | **OpenClaw + 로컬 Hermes/Qwen** |
| 메시징 비서(품질 우선) | OpenClaw + Claude/GPT |
| NousResearch 생태계 자율 에이전트 | Hermes Agent + Hermes |
| 통제된 커스텀 에이전트(앱 내장) | 직접 구현([[Agent-Harness]]) + function calling 모델 |
| 코딩 에이전트(IDE/터미널) | Claude Code / Codex / cmux (별개 영역) |

> [!warning] 자율 런타임의 위험
> OpenClaw류는 **셸·파일·웹에 실제 액션**을 한다 → 권한·샌드박싱·승인 게이트 필수. 자가개선(스스로 코드 작성)은 강력하지만 통제되지 않으면 위험. 최소 권한·격리 환경에서 운용.

---

## 8. 정리

```
모델(두뇌) vs 런타임(몸체)를 먼저 구분하라.
  OpenClaw·Hermes Agent = 런타임 (LLM 갈아끼움, 액션·메모리·자가개선)
  Hermes/Qwen/Claude    = 모델 (런타임에 꽂는 두뇌)

"Hermes 모델로 OpenClaw 대체" = 범주 오류 → 오히려 조합
"Hermes Agent vs OpenClaw"   = 런타임 경쟁재, 용도로 선택
가장 실용: OpenClaw(런타임) + 로컬 Hermes/Qwen(두뇌) = 무료·프라이버시 자율 비서
공통 주의: 실제 액션 실행 → 권한·샌드박싱·승인 게이트 필수
```

---

## 참고문헌
[1] OpenClaw — Wikipedia / openclaw.ai / github.com/openclaw/openclaw (P. Steinberger, 2025.11, MIT)
[2] "What is OpenClaw?" — DigitalOcean, 2026
[3] NousResearch hermes-agent — github.com/nousresearch/hermes-agent (2026.02, MIT)

---

## 관련
- [[Hermes-Agent]] — Hermes 모델 function calling + Hermes Agent 프레임워크
- [[Local-LLM-API-Deployment]] — 로컬 모델을 OpenAI 호환 API로 (런타임 백엔드)
- [[Agent-Harness]] — 직접 구현 시 에이전트 루프·가드레일
- [[MCP-Skill]] — 도구 연결(MCP), 스킬 시스템
- [[Open-Source-Models]] — 백엔드 모델 선택(Hermes/Qwen/Llama)
- [[Security-Checklist]] — 자율 에이전트 권한·샌드박싱
- [[Agent-Security]] — 에이전트 전용 보안: 인젝션·도구남용 방어, 자가개선 통제
- [[Multi-Agent-Coding-Team]] — OpenClaw+역할별 모델로 구성한 코딩 팀 적용 사례
- [[Hermes-Runtime-Internals]] — 런타임 자기관리(서브에이전트·압축·가드레일·샌드박스)
