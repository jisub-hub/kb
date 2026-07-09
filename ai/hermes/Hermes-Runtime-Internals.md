---
tags:
  - ai
  - agent
  - hermes
  - runtime
  - context-management
created: 2026-06-19
---

# Hermes 내부 런타임 메커니즘 (서브에이전트·압축·가드레일·샌드박스)

> [!summary] 한 줄 요약
> 에이전트가 장시간 자율로 돌려면 **스스로를 관리하는 4가지 런타임 장치**가 필요하다 — ① `delegate_task` **서브에이전트**(작업 격리·병렬), ② **컨텍스트 압축**(토큰 한계 대응), ③ **tool-loop 가드레일**(무한루프 차단), ④ **터미널 샌드박스**(실행 격리). Hermes(`~/.hermes`)의 실제 구현·기본값 기준.

> function calling 문법은 [[Hermes-Agent]], 외부 코딩 에이전트 위임은 [[Orchestrating-Claude-Code]], 일반 루프 이론은 [[Agent-Harness]]. 이 노트는 그 사이 빠진 고리 — **Hermes 내부 자기관리**.

---

## 1. delegate_task — 서브에이전트 위임

```
delegate_task = 자식 AIAgent 인스턴스를 띄움
  - 격리된 컨텍스트(별도 대화) + 제한된 툴셋 + 자체 터미널 세션
  - 병렬 배치 기본 최대 3 동시(설정 가능, 하드 상한 없음)
  - delegation.max_iterations: 50 (자식 1개의 루프 한도)
  - 자식의 "최종 요약"만 부모 컨텍스트로 회수 → 부모 컨텍스트 절약
```

```python
delegate_task(tasks=[
    {"goal": "Research topic A", "toolsets": ["web"]},
    {"goal": "Fix the build",   "toolsets": ["terminal", "file"]},
])
```

> [!danger] ⚠️ 서브에이전트는 부모 대화를 **전혀** 모른다
> 자식은 **완전히 새 대화**로 시작 — 부모의 히스토리·이전 툴 호출을 0으로 본다. 자식이 아는 건 오직 `goal`과 `context` 필드에 넣어준 것뿐.
> → 파일 경로·에러 전문·환경(Python 버전 등)을 **전부 명시**해야 함. "Fix the error"(❌) vs "Fix the TypeError in api/handlers.py, line 47, ... 프로젝트는 /home/user/myproject"(✅).

**언제 위임하나**: 병렬 리서치, 격리가 필요한 큰 하위작업, 부모 컨텍스트를 더럽히지 않고 탐색할 때. 위임 비용(자식 부팅·컨텍스트 재구성)이 있으므로 잔작업까지 쪼개지 말 것. → 역할 분담 설계는 [[Multi-Agent-Coding-Team]].

---

## 2. 컨텍스트 압축 — 토큰 한계 대응

긴 세션에서 대화가 컨텍스트 윈도를 넘기 전에 **중간을 요약**한다.

```
compression (config 기본값):
  enabled: true
  threshold: 0.5        # 컨텍스트의 50% 차면 압축 발동
  target_ratio: 0.2     # 압축 후 목표 = 임계 토큰의 20%로 축소
  protect_first_n: 3    # 앞 3개 메시지 보존(시스템·초기 지시 = 정체성)
  protect_last_n: 20     # 최근 20개 보존(진행 중 맥락)
```

```
[보호: 앞 3개] [········ 중간: 요약 압축 ········] [보호: 최근 20개]
   초기 지시·목표                 오래된 왕복             현재 작업 흐름
```
- **왜 앞뒤를 보호하나**: 앞 = 정체성·목표(잃으면 표류), 뒤 = 현재 작업 맥락(잃으면 직전 행동 망각). 위험한 건 중간의 오래된 왕복 → 요약으로 압축.
- 트레이드오프: 압축은 **정보 손실**이다. 압축된 구간의 세부(정확한 수치·경로)는 사라질 수 있으니, 영구적으로 필요한 사실은 메모리/파일에 적어둬야([[Multi-Tenant-Memory-Agent]] 메모리 계층).

---

## 3. tool-loop 가드레일 — 무한루프 차단

에이전트가 같은 실패를 반복하거나 진전 없이 도는 것을 카운터로 잡는다.

```
tool_loop_guardrails (config 기본값):
  warnings_enabled: true        # 경고는 켜고
  hard_stop_enabled: false      # 강제 중단은 기본 끔(보수적)

  warn_after:      { exact_failure: 2, same_tool_failure: 3, idempotent_no_progress: 2 }
  hard_stop_after: { exact_failure: 5, same_tool_failure: 8, idempotent_no_progress: 5 }
```

| 카운터 | 의미 |
|--------|------|
| **exact_failure** | 같은 호출이 같은 에러로 반복 실패 |
| **same_tool_failure** | 한 툴이 (인자 달라도) 계속 실패 |
| **idempotent_no_progress** | 같은 결과만 반복(상태 변화 없음 = 헛돌기) |

- 동작: 임계 도달 시 먼저 **경고**(에이전트에게 "다른 접근을 하라" 신호), 더 누적되면(활성 시) **하드 스톱**.
- 기본이 `hard_stop_enabled: false`인 이유: 정당한 재시도를 너무 일찍 끊으면 작업이 실패한다 → 경고로 유도하고 중단은 신중히. 무인 자동 환경이라면 켜는 걸 고려.
- 일반 가드레일 이론(루프 한도·예산)은 [[Agent-Harness]], 보안 관점은 [[Agent-Security]].

---

## 4. 터미널 샌드박스 — 실행 격리

```
terminal (config 기본값):
  backend: local            # 또는 container
  container_cpu: 1
  container_memory: 5120    # MB
  container_disk: 51200     # MB
  container_persistent: true
  lifetime_seconds: 300     # 세션 수명
  timeout: 180              # 명령 1회 타임아웃
  home_mode: auto
code_execution:
  max_tool_calls: 50
  timeout: 300
```
- **자원 상한**(cpu/mem/disk)으로 폭주·자원 고갈을 가둔다 → K8s 리소스 거버넌스와 같은 사고([[Resource-Management]]).
- `lifetime_seconds`·`timeout`은 **외부 위임의 함정과 동일** — 긴 작업이 잘림([[Orchestrating-Claude-Code]] §3.2와 같은 한도).
- 위험 명령은 승인 게이트를 거침(무인 시 자동 승인은 격리 안에서만) → [[Agent-Security]].

---

## 5. 정리 — 자기관리 4종 세트

```
서브에이전트 : 작업을 격리·병렬화하고 부모 컨텍스트를 지킴 (단, 자식은 백지)
컨텍스트 압축 : 토큰 한계를 넘기 전 중간을 요약 (앞뒤 보호, 손실 주의)
가드레일      : 같은 실패/무진전 반복을 카운터로 차단 (경고 우선, 중단 신중)
샌드박스      : 자원·수명·타임아웃으로 실행을 가둠
→ 이 4가지가 "장시간 자율 운영"의 안정성을 만든다. 하나라도 없으면 표류·폭주·OOM.
```

---

## 6. 관련
- [[Hermes-Agent]] — function calling·툴 호출 문법
- [[Orchestrating-Claude-Code]] — 외부 코딩 에이전트 위임(내부 delegate_task와 대비)
- [[Multi-Agent-Coding-Team]] — 서브에이전트 역할 분담 설계
- [[Agent-Harness]] — 에이전트 루프·가드레일 일반 이론
- [[Agent-Security]] — 샌드박싱·승인 게이트·무인 실행 통제
- [[Multi-Tenant-Memory-Agent]] — 압축 손실을 보완하는 영속 메모리 계층
- [[Autonomous-Agent-Runtimes]] — 런타임 비교 맥락
- [[Tool-Search]] — 툴 스키마의 컨텍스트 비용을 점진적 공개로 줄임(컨텍스트 관리 형제)
- [[Agent-Kanban]] — delegate_task(in-process·휘발) vs durable 보드 조율(대비 프리미티브)
