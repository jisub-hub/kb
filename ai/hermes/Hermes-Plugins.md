---
tags:
  - ai
  - agent
  - hermes
  - plugins
  - extensibility
created: 2026-06-19
---

# Hermes 플러그인 시스템 (커스텀 도구·확장)

> [!summary] 한 줄 요약
> **코어 코드를 고치지 않고** 커스텀 **도구(tool)·훅(hook)·통합**을 에이전트에 붙이는 1차 확장 경로. `~/.hermes/plugins/<name>/`에 `plugin.yaml` + Python을 떨궈 두면 시작 시 자동 로드되어 내장 툴과 나란히 모델에 노출된다. skill·hook·MCP가 모두 그 위에서 등록되는 **확장의 근본 메커니즘**. (`~/.hermes` 기준)

> 훅만 다루는 [[Agent-Event-Hooks]], 지식 문서인 [[Hermes-Skills]], 외부 서버 연결 [[MCP-Skill]]의 **공통 토대**. "내 도구를 어떻게 붙이나"의 답.

---

## 1. 언제 플러그인인가

```
"나/팀/한 프로젝트를 위한 커스텀 툴을 만들고 싶다" → 플러그인이 보통 정답
구분:
  플러그인           = 사용자 정의 도구/훅/통합 (~/.hermes/plugins/, 코어 불변)
  코어 툴(개발자용)  = tools/·toolsets.py에 내장 (Hermes 본체 수정 — 일반 사용자 영역 아님)
```

## 2. 구조 — 디렉터리 한 개

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml      # 매니페스트(name·version·description)
├── __init__.py      # register(ctx) — 스키마↔핸들러 배선의 진입점
├── schemas.py       # 툴 스키마 (LLM이 보는 것: 이름·설명·파라미터)
└── tools.py         # 툴 핸들러 (실제 호출 시 실행되는 코드)
```
- 시작 시 자동 로드 → 툴이 **내장 툴과 나란히** 모델 tools 배열에 등장 → 즉시 호출 가능.

## 3. register(ctx) — 배선 진입점

```python
# __init__.py
def register(ctx):
    # 툴 등록: LLM이 보는 schema ↔ 실행 handler 연결
    ctx.register_tool(schema, handler)
    # 훅 등록: 생명주기 지점에 콜백 (pre_tool_call 차단·pre_llm_call 주입 등)
    ctx.register_hook("pre_tool_call", my_observer)
```
- **스키마 = 모델이 보는 인터페이스**(무엇을·어떤 인자로 부를지), **핸들러 = 실제 동작**. 둘을 분리.
- 같은 `register()` 안에서 툴과 훅을 함께 등록 → 플러그인 하나가 "도구 + 그 도구의 관측/통제"를 묶을 수 있음. (훅 상세 → [[Agent-Event-Hooks]] plugin hooks)

## 4. 확장 3종 — plugin vs skill vs MCP ⭐

| | **Plugin** | **Skill** | **MCP** |
|---|-----------|-----------|---------|
| 무엇 | 실행 코드(도구·훅) 추가 | 지식 문서(절차) | 외부 서버의 도구 연결 |
| 위치 | `~/.hermes/plugins/` (인프로세스 Python) | `~/.hermes/skills/` (md) | 원격/로컬 MCP 서버 |
| 모델에 보이는 것 | 새 툴 스키마 | (로드 시) 절차 텍스트 | 서버가 노출한 툴 |
| 실행 | 플러그인 핸들러가 직접 | 모델이 일반 툴로 수행 | MCP 서버가 |
| 쓸 때 | 내 로직/통합을 코드로 | 재사용 워크플로 지식 | 남이 만든 도구 붙이기 |

```
정리: "코드로 새 동작" = Plugin / "매뉴얼" = Skill([[Hermes-Skills]]) / "외부 도구 연결" = MCP([[MCP-Skill]])
이 셋이 다수가 되면 컨텍스트 비용↑ → [[Tool-Search]] progressive disclosure로 완화
```

## 5. 특성·신뢰 모델

```
인프로세스 실행: 플러그인 코드는 에이전트와 같은 프로세스에서 돈다(훅도 in-process)
  → 빠르고 상태 공유 쉬우나, 격리 없음 → 신뢰하는 코드만(디렉터리 신뢰 모델)
  ⚠️ 서브프로세스 격리가 필요하면 shell hook을 쓰는 게 안전([[Agent-Event-Hooks]] 비교표)
보안: 커스텀 툴이 임의 동작을 하므로 권한·감사는 [[Agent-Security]] 원칙 적용
```

## 6. 확장 메커니즘 전체 지도

```
Plugin (코드)  ─┐
Skill (지식)   ─┼─→ 에이전트 능력 확장
MCP (외부도구) ─┘
Hook (생명주기) → 위 동작을 관측·차단·증강 (plugin이 흔히 함께 등록)
Tool-Search    → 늘어난 도구의 컨텍스트 비용 관리
→ 플러그인은 이 중 "직접 코드로 도구·훅을 추가"하는 가장 기본 경로.
```

---

## 7. 관련
- [[Agent-Event-Hooks]] — 플러그인이 등록하는 훅(3계층 중 plugin hooks)·shell hook 격리 비교
- [[Hermes-Skills]] — 지식 문서 확장(plugin=코드 vs skill=매뉴얼)
- [[MCP-Skill]] — 외부 MCP 서버 도구 연결(plugin=인프로세스 vs MCP=원격)
- [[Tool-Search]] — 도구가 많아질 때 컨텍스트 비용 점진적 공개
- [[Agent-Security]] — 커스텀 도구의 권한·감사·신뢰 모델
- [[Hermes-Agent]] — function calling·툴 호출 기본
