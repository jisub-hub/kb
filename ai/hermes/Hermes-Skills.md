---
tags:
  - ai
  - agent
  - hermes
  - skills
  - context-management
created: 2026-06-19
---

# Hermes Skills — 호출(로딩) 메커니즘과 절차 기억

> [!summary] 한 줄 요약
> Skill은 **함수처럼 "호출"되는 게 아니라, 필요할 때 펼쳐 읽는 on-demand 지식 문서**다. 평소엔 이름·설명 목록만 보이고(progressive disclosure), 모델이 필요하다고 판단하면 `skill_view`로 본문을 로드해 그 절차대로 일반 툴로 작업한다. 더 나아가 에이전트는 `skill_manage`로 **자기 워크플로를 skill로 저장**한다 — 즉 skill = 에이전트의 **절차 기억**. (`~/.hermes` 기준)

> 점진적 공개의 툴 버전은 [[Tool-Search]], MCP 도구 연결은 [[MCP-Skill]], 선언적 기억(사실)은 [[Agent-Persistent-Memory]]. 이 노트는 **"매뉴얼을 어떻게 펼치고 누가 쓰나"**.

---

## 1. 핵심 구분 — "호출"이 아니라 "로딩"

```
MCP 툴  = 실행 가능한 함수 (call → 결과)
Skill   = 그때그때 읽는 매뉴얼/절차 문서 (load → 모델 컨텍스트에 주입 → 모델이 일반 툴로 수행)
→ skill 자체엔 실행 코드가 없다. "이렇게 하라"는 지시문. 실행은 모델이 terminal·read_file 등으로.
```
- 모든 skill은 `~/.hermes/skills/`에 산다(단일 진실 소스). 번들·허브 설치·에이전트 자작 모두 여기로.
- agentskills.io 오픈 표준 호환.

## 2. Progressive Disclosure — 3단계 로딩 (토큰 절약)

```
Level 0: skills_list()           → [{name, description, category}, ...]   (~3k 토큰, 목록만 상시)
Level 1: skill_view(name)        → 그 skill 본문 + 메타데이터              (필요 시)
Level 2: skill_view(name, path)  → skill 안의 특정 참조 파일만             (더 필요 시)
```
> 평소엔 **이름+설명만** 시스템에 노출 → 모델이 "필요하다" 판단할 때만 본문을 펼친다. [[Tool-Search]]가 *툴 스키마*에 쓰는 것과 **같은 패턴**(대상이 지식 문서일 뿐).

## 3. 누가 트리거하나 — 4경로

```
① 모델 자율    : skills_list로 목록 보고 → 필요한 것 skill_view (description·When to Use가 선택 신호)
② 슬래시 명령  : /plan, /research → 그 skill 강제 로드(사람 지시)
③ 번들        : /<bundle-name> → YAML에 묶인 여러 skill 한꺼번에 로드(+뒤 텍스트는 지시로 첨부)
④ 조건부 자동  : requires_toolsets / fallback_for_toolsets 메타로 상황 따라 노출/은닉(§5)
```
- cron 작업에도 skill을 붙인다(`--skill`) → 예약 실행이 절차지식을 들고 동작. [[Agent-Scheduled-Execution]]
- Kanban worker·delegate_task에도 toolset/skill을 좁혀 줄 수 있음. [[Hermes-Runtime-Internals]]

## 4. SKILL.md 형식 — 선택 신호가 핵심

```markdown
---
name: my-skill
description: Brief description     # ← Level 0 목록에 노출 → 모델이 이걸 보고 고름 (가장 중요)
platforms: [macos, linux]         # 선택: OS 제한
metadata:
  hermes:
    requires_toolsets: [terminal]   # 조건부 활성(§5)
    config: [{key: my.setting, ...}]# config.yaml 연동(로드 시 주입)
---
# Skill Title
## When to Use       # ← "언제 쓰는지" — 모델의 두 번째 선택 신호
## Procedure         # ← 로드되면 이 절차가 모델 컨텍스트로
```
> 잘 동작하게 하려면 **`description`과 `When to Use`를 명확히** 써야 한다 — 모델이 로드 여부를 이 둘로 판단한다(애매하면 안 불려나오거나 엉뚱하게 불려나옴).

## 5. 조건부 활성 (Fallback Skills) ⭐

```
fallback_for_toolsets: [web]   → 해당 툴셋이 "있으면 숨김", 없으면 자동 노출(대체재)
requires_toolsets: [terminal]  → 해당 툴셋이 "있을 때만 노출"
(tools 단위 버전도 있음: fallback_for_tools / requires_tools)

예: 내장 duckduckgo-search 는 fallback_for_toolsets: [web]
    FIRECRAWL_API_KEY 있음 → web 툴셋 활성 → web_search 사용, DDG skill 숨김
    키 없음 → web 툴셋 비활성 → DDG skill 자동 등장(폴백)
```
> 환경에 따라 skill 목록이 동적으로 바뀐다 → 프리미엄 도구 없을 때만 무료/로컬 대안이 뜨게.

## 6. skill_manage — 에이전트의 절차 기억 ⭐

```
에이전트는 skill_manage 툴로 자기 skill을 생성·수정·삭제(patch/edit/write_file/remove_file/delete)
  → "비자명한 워크플로를 알아냈을 때 그 방법을 skill로 저장 → 다음에 재사용"
  = 절차 기억(procedural memory)
config: skills.creation_nudge_interval(기본 15) — 주기적으로 skill화를 유도
대비: [[Agent-Persistent-Memory]]는 "사실"의 선언적 기억(MEMORY.md), skill은 "방법"의 절차 기억
```

```
선언적 기억(MEMORY.md/USER.md) : 무엇을 안다 (환경·선호·사실)        — 항상 주입, 작음
절차 기억(skills)              : 어떻게 한다 (재사용 워크플로)        — on-demand 로드, 큼
세션 기록(session_search)      : 무슨 일이 있었나 (과거 대화)         — 검색
→ 세 기억이 합쳐져 "세션을 넘는 학습"을 이룬다.
```

## 7. 로드 시 동작 / 보안

```
config 연동: skill의 config 값(skills.config)은 로드 시 컨텍스트에 자동 주입(에이전트가 설정값 인지)
시크릿: 누락된 값은 "로컬 CLI에서 skill이 실제 로드될 때만" 안전하게 요청
        메신저 표면에선 시크릿을 채팅으로 묻지 않음 → hermes setup / ~/.hermes/.env 안내
```

---

## 8. 관련
- [[Tool-Search]] — 같은 progressive disclosure를 *툴 스키마*에 적용(형제 메커니즘)
- [[Hermes-Plugins]] — 코드로 도구·훅 추가(skill=매뉴얼 vs plugin=실행 코드)
- [[LLM-Wiki]] — description=선택신호·progressive disclosure의 KB판(같은 패턴)
- [[MCP-Skill]] — MCP 도구 연결 + 스킬 시스템 개요
- [[Agent-Persistent-Memory]] — 선언적 기억(사실) vs skill의 절차 기억
- [[Hermes-Runtime-Internals]] — delegate_task worker에 toolset/skill 한정
- [[Agent-Scheduled-Execution]] — cron 작업에 skill 첨부(`--skill`)
- [[Agent-Harness]] — 에이전트 컨텍스트·도구 설계 일반
