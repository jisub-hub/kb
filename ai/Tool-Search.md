---
tags:
  - ai
  - agent
  - hermes
  - mcp
  - context-management
created: 2026-06-19
---

# Tool Search — 툴 과부하와 점진적 공개 (Progressive Disclosure)

> [!summary] 한 줄 요약
> MCP 서버·플러그인을 많이 붙이면 **툴 JSON 스키마가 매 턴 컨텍스트를 잠식**한다(15+ 툴이면 심각). **Tool Search**는 툴을 3개의 브리지 툴로 대체하고 **필요한 스키마만 on-demand로 로드**하는 점진적 공개 레이어. Hermes(`~/.hermes`) 기준이며, Claude Code의 deferred tools/ToolSearch와 동형 개념.

> MCP 연결 자체는 [[MCP-Skill]], 컨텍스트 관리 일반은 [[Hermes-Runtime-Internals]], 입력 토큰 비용은 [[Prompt-Caching]].

---

## 1. 문제 — 툴 스키마가 컨텍스트를 먹는다

```
모델에게 보이는 tools 배열 = 모든 툴의 JSON 스키마(이름·설명·파라미터)
  툴 N개 → 매 턴 N개 스키마를 입력에 동봉
  MCP 서버 여러 개(GitHub·Slack·DB·...) = 수십 개 툴 → 수천~수만 토큰 상시 점유
영향: ① 컨텍스트 낭비(긴 대화 여지↓) ② 입력 비용↑ ③ 선택 정확도↓
      (관련 없는 툴이 많을수록 모델이 헷갈림)
```

## 2. 해법 — 3개 브리지 툴로 대체

활성화되면 deferrable 툴들이 사라지고 **3개 브리지 툴**만 노출된다:

```
tool_search(query, limit?)   → deferred 카탈로그 검색(이름·요약만)
tool_describe(name)          → 그 툴 1개의 전체 스키마 on-demand 로드
tool_call(name, arguments)   → 실제 호출
```

```
모델: tool_search("create a github issue")
  → { matches: [{ name: "mcp_github_create_issue", ... }] }
모델: tool_describe("mcp_github_create_issue")
  → { parameters: { ... } }          # 이때만 스키마 로드
모델: tool_call("mcp_github_create_issue", { title, body })
  → { ok: true, issue_number: 42 }
```
> 매 턴 비용이 "전체 툴 스키마" → "3개 브리지 스키마(~300토큰)"로 고정된다. 검색→설명→호출의 **왕복 1~2회**가 추가되는 대신.

## 3. 자동 활성 임계 (auto)

```
기본 enabled: auto
  deferrable 툴 스키마가 컨텍스트의 ≥ threshold_pct(기본 10%)를 차지할 때만 활성
  매번 tools 배열을 만들 때 재평가 →
    - 툴 몇 개뿐 + 긴 컨텍스트 모델 → 절대 활성 안 됨(순수 pass-through, 오버헤드 0)
    - MCP 15+ → 활성 시작
    - 세션 중 MCP 제거 → 다음 어셈블리에 직접 노출로 복귀
```

## 4. 코어 툴은 항상 직접 노출 ⚠️

```
deferral 대상: MCP 툴 + 비코어 플러그인 툴만
항상 직접 노출(_HERMES_CORE_TOOLS): terminal·read_file·write_file·patch·
  search_files·todo·memory·web_search·execute_code·delegate_task·
  session_search·clarify ... (핵심 능력셋)
→ 매 턴 쓰는 코어 도구를 검색 뒤로 숨기면 오히려 느려지므로 제외
```

## 5. 브리지 언랩 — 보안/훅은 실제 툴 기준 ⚠️

```
tool_call 실행 시 Hermes가 브리지를 "언랩"하고 실제 툴을 직접 호출한 것처럼 디스패치
  → pre/post 훅·tool-loop 가드레일·승인 프롬프트가 모두 "실제 툴명" 기준으로 동작
  → CLI/게이트웨이 활동 피드도 언랩되어 실제 툴이 보임(tool_call로 안 보임)
즉, 점진적 공개가 [[Agent-Security]] 승인·감사를 우회하지 않는다.
```

## 6. 설정 & 트레이드오프

```yaml
tools:
  tool_search:
    enabled: auto          # auto(기본) | on(항상) | off
    threshold_pct: 10      # auto 활성 임계(%) 0–100
    search_default_limit: 5
    max_search_limit: 20   # 1–50
  # tool_search: true       # 레거시 불리언 = {enabled: auto}
```

> [!warning] 언제 쓰지 말까
> Tool Search는 **고정 비용(브리지 스키마 ~300토큰 + 왕복 1~2회)**을 치르고 deferred 스키마를 절약한다. 툴이 적으면 절약분 < 고정비용 → 손해. 그래서 기본 `auto`가 임계(10%) 미만에선 끄는 것. **소수 툴 세션엔 켜지 마라.**

```
켤 가치 O: MCP 서버 여럿·툴 수십 개·짧은 컨텍스트 모델
켤 가치 X: 코어 툴 위주·툴 몇 개·긴 컨텍스트 모델(→ auto가 알아서 pass-through)
```

---

## 7. 관련
- [[MCP-Skill]] — MCP 서버/툴 연결(여기서 툴 수가 늘어남)
- [[Hermes-Runtime-Internals]] — 컨텍스트 압축·서브에이전트 등 런타임 자기관리
- [[Hermes-Skills]] — 같은 progressive disclosure를 *지식 문서(skill)*에 적용(형제)
- [[Prompt-Caching]] — 입력 토큰 비용 절감(보완 관계: 캐싱 vs 미노출)
- [[Agent-Security]] — 브리지 언랩이 승인·감사를 우회하지 않음
- [[Agent-Harness]] — 툴 설계·선택 일반론
