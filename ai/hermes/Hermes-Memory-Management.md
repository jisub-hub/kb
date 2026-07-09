---
tags:
  - ai
  - agent
  - hermes
  - memory
  - sessions
created: 2026-06-24
---

# Hermes 메모리·세션 관리 (영속 메모리 · 세션 모델 · 컨텍스트 · 멀티테넌트)

> [!summary] 한 줄 요약
> Hermes가 **무엇을·어떻게 기억하고 잊는가**를 한곳에 정리. ① 세션을 넘는 **영속 메모리**(`MEMORY.md`/`USER.md`, 용량 제한·큐레이션·frozen snapshot), ② **세션 모델**(per-user/platform 격리·리셋·동시 세션), ③ **세션 내 컨텍스트 관리**(auto-compact 안 함 + 대화 압축), ④ **멀티테넌트 메모리**(개인/팀 KB 누적·RLS 격리·memory poisoning 방어). 출처: [[Agent-Persistent-Memory]] · [[Multi-Tenant-Memory-Agent]] · [[Hermes-Runtime-Internals]] · [[Hermes-Deployment-Surface]].

> 시간 축으로 정리하면 — **세션 내**(휘발: 대화 압축) → **세션 간**(영속: MEMORY.md/USER.md) → **테넌트 간**(누적: KB+RLS). 아래 각 절이 이 세 층을 다룬다.

---

## 1. 영속 메모리 파일 — MEMORY.md / USER.md

세션을 넘어 스스로 기억하는 **바운디드(용량 제한)·큐레이션** 된 두 파일. RAG(외부 검색)도 파인튜닝(가중치)도 아닌, **"프롬프트에 들고 다니는 작은 영속 노트"**. ([[Agent-Persistent-Memory]])

| 파일 | 용도 | 용량 한도 |
|------|------|----------|
| **MEMORY.md** | 에이전트의 개인 노트 — 환경 사실·관례·학습한 것 | 2,200자 (~800토큰) |
| **USER.md** | 사용자 프로필 — 선호·소통 스타일·기대 | 1,375자 (~500토큰) |

- 저장: `~/.hermes/memories/`. 세션 시작 시 **시스템 프롬프트에 주입**.
- **용량 한도가 핵심 설계** — 무한히 쌓지 않음 → 항상 "지금 가장 중요한 것"만 유지하도록 강제.
- 시스템 프롬프트 헤더에 **용량 %·문자수**가 노출(예: `[67% — 1,474/2,200 chars]`)되어 에이전트가 남은 공간을 스스로 인지. 항목은 `§` 구분자로 분리.

### 1.1 memory 툴 — add / replace / remove (read는 없다)

```
add     : 새 항목 추가
replace : 기존 항목 교체(old_text substring 매칭) — 갱신·통합
remove  : 무의미해진 항목 삭제(old_text substring 매칭)
read 없음 : 메모리는 이미 시스템 프롬프트에 주입돼 있어 "보는" 행위가 불필요
```

### 1.2 Frozen Snapshot — prefix cache를 지키는 의도된 설계 ⭐

> [!important] 세션 시작 시 한 번 새기고(frozen), 세션 중에는 변하지 않는다
> 세션 중 add/remove 해도 **시스템 프롬프트엔 즉시 반영되지 않는다**(다음 세션부터). 단, 디스크에는 즉시 저장되고, 툴 응답은 항상 live 상태를 보여준다.

- **왜?** 시스템 프롬프트가 세션 중 바뀌면 prefix cache가 깨진다 → 고정해서 [[Prompt-Caching]]의 입력 KV 재사용을 보존(성능·비용).
- 트레이드오프 핵심: **실시간 반영 vs 캐시 보존** → 캐시 보존을 택함.

### 1.3 큐레이션 원칙 — 가득 차면 에러로 강제 정리

> [!warning] auto-compact 안 함 — 조용히 버리지 않는다
> 영속 메모리는 한도 초과 시 **자동 요약/삭제하지 않고 에러를 던져** 에이전트가 직접 무엇을 버릴지 의식적으로 결정하게 한다(자동 손실 ❌). 망각을 능동적 결정으로 만드는 것이 큐레이션의 핵심.

```
add가 한도를 넘기면 → success:false + "Consolidate now: replace로 병합 또는 remove,
                       then retry this add — all in this turn."
절차: ① 현재 항목 확인 → ② 통합/삭제 대상 식별 → ③ replace로 짧게 병합 → ④ 다시 add
⚠️ replace도 한도에 묶임(더 긴 내용으로 바꾸면 또 넘침)
베스트: 80% 넘으면(헤더에 보임) 새로 넣기 전에 미리 통합
```

> 세 번째 기둥은 **session_search**(과거 대화 검색): 작고 항상 켜진 메모리(MEMORY.md/USER.md) + 큰 이력은 필요 시 검색. 둘 다 **비파라메트릭**(가중치를 안 건드림).

---

## 2. 세션 모델 — per-user/platform 격리·리셋·동시 세션

배포 표면(API 서버·메신저 게이트웨이)에서 세션이 어떻게 갈리는가. ([[Hermes-Deployment-Surface]] §3)

| 설정 | 기본값 | 의미 |
|------|--------|------|
| `group_sessions_per_user` | `true` | 사용자별 세션 분리 — 대화·메모리 격리 |
| `max_concurrent_sessions` | `null` | 동시 세션 상한(기본 무제한) |
| `session_reset` | idle / at_hour | 유휴 시간 또는 지정 시각 기준 세션 리셋 |

- **플랫폼·user별로 세션이 갈리고, 각 세션이 영속 메모리를 가진다** → per-platform 메모리([[Agent-Persistent-Memory]]).
- 멀티 프로필 게이트웨이에서는 **profile = 독립된 봇 토큰·세션·메모리·정체성 묶음**으로, 여러 named 봇이 각자 상주하며 서로 다른 세션·메모리를 가진다([[Hermes-Deployment-Surface]] §4).
- session_reset이 발동하면 새 세션이 시작되며, 영속 메모리는 다시 frozen snapshot으로 주입된다(§1.2).

---

## 3. 컨텍스트 관리 — 세션 내 압축 (휘발)

세션 **내부**에서 대화가 길어질 때 토큰 한계를 넘기 전 중간을 요약한다. 이는 §1의 영속 메모리(세션 간)와 달리 **세션 한정·휘발**이다. ([[Hermes-Runtime-Internals]] §2)

```
compression (config 기본값):
  enabled: true
  threshold: 0.5        # 컨텍스트의 50% 차면 압축 발동
  target_ratio: 0.2     # 압축 후 목표 = 임계 토큰의 20%로 축소
  protect_first_n: 3    # 앞 3개 메시지 보존(시스템·초기 지시 = 정체성)
  protect_last_n: 20    # 최근 20개 보존(진행 중 맥락)
```

```
[보호: 앞 3개] [········ 중간: 요약 압축 ········] [보호: 최근 20개]
   초기 지시·목표                 오래된 왕복             현재 작업 흐름
```

- **왜 앞뒤를 보호하나**: 앞 = 정체성·목표(잃으면 표류), 뒤 = 현재 작업 맥락(잃으면 직전 행동 망각). 위험한 건 중간의 오래된 왕복 → 요약으로 압축.

> [!warning] 두 종류의 "compact"를 혼동하지 말 것
> - **세션 내 대화 압축**(이 절): 자동(threshold 발동)·**정보 손실** 있음·휘발. 압축 구간의 정확한 수치·경로는 사라질 수 있다.
> - **영속 메모리**(§1.3): **auto-compact 안 함** — 가득 차면 에러로 에이전트가 직접 큐레이션.
> → 영구적으로 필요한 사실은 압축에 맡기지 말고 메모리/파일에 적어둬야 한다([[Agent-Persistent-Memory]] · [[Multi-Tenant-Memory-Agent]] 메모리 계층).

---

## 4. 멀티테넌트 메모리 — 개인/팀 KB 누적·격리

사용자/팀 인증 → Hermes 호출 → 상호작용을 **개인·팀 지식베이스(KB)에 누적** → 다음 호출 때 검색·주입하여 점차 개인화. **모델은 그대로, KB가 자란다**(RAG 메모리이지 파인튜닝이 아님). ([[Multi-Tenant-Memory-Agent]])

### 4.1 메모리 계층

```
단기(세션):   현재 대화 히스토리 → 일정 길이 후 요약하여 장기로 (§3 압축과 연결)
장기(영속):
  - fact/preference : "이 사용자는 Spring을 쓴다" (구조화, 검색·주입)
  - episode         : 과거 상호작용 임베딩 (RAG 검색)
  - team            : 팀 공유 KB (문서·결정사항)
→ 호출 시 관련된 것만 선택 주입 (전부 X — 토큰·노이즈)
```

### 4.2 RLS 격리 — 데이터 유출 = 치명적

> [!warning] 격리는 앱 로직이 아니라 DB 레벨로 강제
> 한 사용자/팀의 기억이 다른 테넌트로 새면 안 된다.

```
- 모든 KB 레코드에 tenant_id (user_id / team_id) 컬럼
- PostgreSQL Row-Level Security(RLS) 또는 쿼리 강제 필터
- 접근 키는 JWT의 tenant_id에서만 (앱이 임의 지정 금지)
- 개인 KB(user_id) + 팀 공유 KB(team_id) 계층 분리
```
→ 인증 [[OAuth2-JWT]], 벡터 [[pgvector]] · [[Embedding]], 검색 [[RAG]].

### 4.3 Memory Poisoning — 위험과 방어 ⚠️

> [!danger] 저장된 기억이 미래 행동을 조종한다
> 악의적 입력이 KB에 저장되면 다음 호출을 오염시켜 인젝션이 **지속화**된다. 자율 에이전트 보안의 핵심.

| 위험 | 설명 | 대응 |
|------|------|------|
| **테넌트 유출** | 기억이 다른 사용자에게 노출 | RLS·tenant 필터 강제(§4.2) |
| **Memory Poisoning** | 악의적 입력이 KB 저장→다음 호출 오염 | 저장 전 검증, 사용자발화/시스템사실 **신뢰 경계 분리** |
| **메모리 노이즈** | 다 저장하면 검색 품질↓ | 중요도 점수·요약·중복 제거·선별 저장 |
| **기억 충돌** | 상반된 기억 누적 | 최신 우선·신뢰도 가중·주기적 정리 |
| **프라이버시** | 민감정보·삭제권(GDPR) | 보존기간·삭제 API·감사 로그 |
| **비용** | 매 호출 검색+주입→토큰↑ | 프롬프트 캐싱·요약 메모리 |

> 가드레일·승인 게이트는 [[Agent-Security]], [[Agent-Harness]].

---

## 5. 운영 팁 — 메모리 비대화 방지·큐레이션·중복 제거

```
영속 메모리(§1):
  - 80% 넘으면(헤더 % 표시) 새 add 전에 replace로 미리 통합
  - 적고 안정적인 사실(정체성·선호·환경)에만 사용 — 큰 이력은 session_search/RAG로
  - replace로 겹치는 항목 병합, 무의미해진 항목은 remove

멀티테넌트 KB(§4):
  - 선별 저장(worthRemembering) — 다 저장하면 노이즈↑·검색 품질↓
  - 중요도 점수·요약 후 저장·중복 제거
  - 상반 기억은 최신 우선·신뢰도 가중·주기적 정리
  - 저장 전 검증으로 poisoning 차단(사용자 발화 ≠ 시스템 사실)

세션 내 압축(§3):
  - 압축은 손실 → 영구 필요 사실은 메모리/파일로 빼두기
  - 앞 3 / 뒤 20 보호로 정체성·현재 맥락은 유지됨
```

---

## 6. 관련
- [[Agent-Persistent-Memory]] — MEMORY.md/USER.md·frozen snapshot·큐레이션(§1·§5 출처)
- [[Hermes-Runtime-Internals]] — 세션 내 컨텍스트 압축(§3 출처)·tool-loop·서브에이전트
- [[Hermes-Deployment-Surface]] — per-user/platform 세션·멀티 프로필(§2 출처)
- [[Multi-Tenant-Memory-Agent]] — 개인/팀 KB·RLS·memory poisoning(§4 출처)
- [[Agent-Security]] — 가드레일·승인 게이트·인젝션 방어
- [[Prompt-Caching]] — frozen snapshot이 보존하려는 prefix cache
- [[RAG]] · [[pgvector]] · [[Embedding]] — 외부 검색 지식·벡터 저장
- [[OAuth2-JWT]] — 인증·테넌트 식별
