---
tags:
  - ai
  - agent
  - hermes
  - memory
  - context-management
created: 2026-06-19
---

# 에이전트 영속 메모리 (MEMORY.md / USER.md 큐레이션)

> [!summary] 한 줄 요약
> 에이전트가 **세션을 넘어 스스로 기억**하는 방법 — 바운디드(용량 제한)·큐레이션(에이전트가 직접 관리)된 두 파일(`MEMORY.md`·`USER.md`)을 세션 시작 시 시스템 프롬프트에 **고정 스냅샷**으로 주입. RAG(외부 검색)도, 파인튜닝(가중치)도 아닌 **"프롬프트에 들고 다니는 작은 영속 노트"**. Hermes(`~/.hermes`) 기준이며 Claude Code의 MEMORY.md와 동형.

> 외부 검색 지식은 [[RAG]], 멀티테넌트 메모리 격리는 [[Multi-Tenant-Memory-Agent]], 세션 내 압축은 [[Hermes-Runtime-Internals]] §2. 이 노트는 **"에이전트가 무엇을·어떻게 기억하나"**의 큐레이션 패턴.

---

## 1. 2파일 구조

| 파일 | 용도 | 용량 한도 |
|------|------|----------|
| **MEMORY.md** | 에이전트의 개인 노트 — 환경 사실·관례·학습한 것 | 2,200자 (~800토큰) |
| **USER.md** | 사용자 프로필 — 선호·소통 스타일·기대 | 1,375자 (~500토큰) |

- 저장: `~/.hermes/memories/`. 세션 시작 시 시스템 프롬프트에 주입.
- **용량 한도가 핵심 설계**: 무한히 쌓지 않음 → 항상 "지금 가장 중요한 것"만 유지하도록 강제.

## 2. Frozen Snapshot — prefix cache를 지키는 의도된 설계 ⭐

```
세션 시작 시 메모리를 디스크에서 읽어 시스템 프롬프트에 "한 번" 새김(frozen)
  → 세션 중에는 변하지 않음
  ⚠️ 세션 중 add/remove 해도 시스템 프롬프트엔 즉시 반영 안 됨(다음 세션부터)
     (단, 디스크에는 즉시 저장 / 툴 응답은 항상 live 상태를 보여줌)

왜 이렇게? 시스템 프롬프트가 세션 중 바뀌면 prefix cache가 깨진다.
  → 고정해서 [[Prompt-Caching]]의 입력 KV 재사용을 보존(성능·비용)
```
> 이 한 가지가 메모리 설계의 트레이드오프 핵심: **실시간 반영 vs 캐시 보존** → 캐시 보존을 택함.

## 3. 시스템 프롬프트에 보이는 형태

```
══════════════════════════════════════
MEMORY (your personal notes) [67% — 1,474/2,200 chars]
══════════════════════════════════════
User's project is a Rust web service at ~/code/myapi using Axum + SQLx
§
This machine runs Ubuntu 22.04, has Docker and Podman installed
§
User prefers concise responses, dislikes verbose explanations
```
- 헤더에 **용량 %·문자수** 노출 → 에이전트가 남은 공간을 스스로 인지.
- 항목은 `§`(구분자)로 분리, 멀티라인 가능.

## 4. memory 툴 — add / replace / remove (read는 없다)

```
add     : 새 항목 추가
replace : 기존 항목 교체(old_text substring 매칭) — 갱신·통합
remove  : 무의미해진 항목 삭제(old_text substring 매칭)
read 액션 없음: 메모리는 시스템 프롬프트에 이미 주입돼 있어 "보는" 행위가 불필요
```

## 5. Auto-compact 안 함 — 가득 차면 에러로 강제 큐레이션 ⚠️

> [!warning] 조용히 버리지 않는다 — 한도 초과 시 **에러를 던져** 에이전트가 직접 정리하게 만든다
> 자동 요약/삭제로 슬그머니 정보를 잃는 대신, 에이전트가 "무엇을 버릴지" 의식적으로 결정하도록 설계.

```
add가 한도를 넘기면:
  { "success": false,
    "error": "Memory at 2,100/2,200. Adding 250 chars would exceed...
              Consolidate now: replace로 겹치는 항목 병합 또는 remove,
              then retry this add — all in this turn.",
    "current_entries": [...], "usage": "2,100/2,200" }

에이전트 절차: ① 현재 항목 확인 → ② 통합/삭제 대상 식별
            → ③ replace로 병합(짧게) → ④ 다시 add
⚠️ replace도 한도에 묶임: 더 긴 내용으로 바꾸면 또 넘침 → 더 줄이거나 다른 항목 제거
베스트: 80% 넘으면(헤더에 보임) 새로 넣기 전에 미리 통합
```
- 이것이 "큐레이션"의 핵심 — **망각을 에이전트의 능동적 결정으로** 만든다(자동 손실 ❌).

## 6. 세 번째 기둥 — Session Search

```
MEMORY.md/USER.md(작고 항상 켜진 메모리) + session_search(과거 대화 검색)
  → 작은 건 상시 프롬프트에, 큰 이력은 필요 시 검색(RAG식)
즉 이 시스템은 "파라메트릭 아님" — 가중치를 안 건드린다([[Parametric-Knowledge-Editing]]와 대비).
  영속 노트 = 프롬프트 주입(비파라메트릭) / RAG = 외부 검색(비파라메트릭) / 둘은 입자만 다름
```

## 7. 다른 지식 방식과의 자리매김

```
영속 메모리(이 노트): 작고·항상·큐레이션된 사실 → 시스템 프롬프트(매 세션)
RAG([[RAG]])        : 크고·검색되는 외부 문서 → 필요 시 컨텍스트
세션 내 압축([[Hermes-Runtime-Internals]] §2): 긴 대화를 요약(휘발, 세션 한정)
파라메트릭 편집([[Parametric-Knowledge-Editing]]): 가중치 자체를 수정(연구 단계)
→ 영속 메모리는 "정체성·선호·환경" 같은 적고 안정적인 사실에 최적
```

---

## 8. 관련
- [[Prompt-Caching]] — frozen snapshot이 보존하려는 prefix cache
- [[Hermes-Runtime-Internals]] — 세션 내 컨텍스트 압축(휘발) vs 영속 메모리
- [[Hermes-Skills]] — 절차 기억(skill_manage) vs 이 노트의 선언적 기억(사실)
- [[Multi-Tenant-Memory-Agent]] — 멀티테넌트 격리·memory poisoning(영속 메모리의 보안면)
- [[RAG]] — 외부 검색 지식(상보: 큰 이력은 session_search/RAG)
- [[Parametric-Knowledge-Editing]] — 가중치 편집(대비: 메모리는 비파라메트릭)
- [[Agent-Harness]] — 에이전트 컨텍스트 관리 일반
- [[LLM-Wiki]] — 메모리판 큐레이션과 같은 원리의 파일 기반 KB 구축
- [[Hermes-Deployment-Surface]] — 플랫폼·user별 세션이 각자 영속 메모리를 가짐
