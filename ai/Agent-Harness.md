---
tags:
  - ai
  - agent
  - harness
  - tool-use
  - mcp
created: 2026-06-15
---

# Agent Harness (에이전트 하니스)

> [!summary] 한 줄 요약
> LLM을 **반복 루프** 안에서 돌리며 *도구 호출 → 실행 → 결과 반영*을 자동화하는 런타임. 모델은 "무엇을 할지"(tool_use)를 내놓고, **하니스가 실제로 그것을 실행**한 뒤 결과를 다시 모델에 돌려준다. 컨텍스트·권한·관측을 책임지는 것이 하니스의 역할.

> 하니스(harness) = 모델을 둘러싼 실행 환경. 루프, 툴 실행, 컨텍스트 관리, 가드레일, 로깅이 전부 여기에 있다. (모델은 텍스트만 내놓는다 — 행동은 하니스가 한다.)

---

## 1. 에이전트 루프 (가장 중요한 구조)
```
사용자 요청
   │
   ▼
┌───────────────────────────────────────┐
│ 1. LLM 호출 (messages + tools)         │◄────────────┐
│ 2. 응답 검사                            │             │
│    - stop_reason == end_turn → 종료     │             │
│    - tool_use 블록이 있으면 ↓           │             │
│ 3. 하니스가 툴 실행 (DB/API/파일...)    │             │
│ 4. tool_result 를 messages에 추가 ──────┼─────────────┘
└───────────────────────────────────────┘
```
- 모델은 **stateless** → 매 반복마다 전체 messages(이전 tool_use/tool_result 포함)를 다시 보낸다.
- `tool_use_id` 와 `tool_result` 는 **1:1로 매칭**되어야 한다.

---

## 2. 에이전트를 만들 것인가? (먼저 판단)
> [!warning]
> 에이전트는 비싸고 느리고 비결정적이다. **단일 호출/워크플로우로 되면 그걸 써라.** ([[_index|티어 1~2]])

4가지 기준 모두 통과할 때만 3티어(에이전트):
- **복잡성**: 다단계이고 사전에 완전 명세 불가
- **가치**: 높은 비용/지연을 정당화
- **실현성**: 모델이 잘하는 작업 유형
- **오류 비용**: 실수를 잡아 복구 가능(테스트/리뷰/롤백)

---

## 3. 툴(tool) 설계
| 구분 | 누가 실행 | 예 |
|------|----------|-----|
| **클라이언트 툴** | **내 하니스** | DB 조회, 사내 API, 파일 쓰기, 결제 |
| **서버 툴** | 제공자 인프라 | 코드 실행 샌드박스, 웹 검색 |

### bash vs 전용 툴
- **bash 하나** → 넓은 능력, 하지만 하니스는 불투명한 명령 문자열만 봄.
- **전용 툴**(타입 있는 인자) → 하니스가 **게이트/렌더/감사/병렬화** 가능.
- 규칙: **넓힐 땐 bash, 통제할 땐 전용 툴로 승격.**
  - 되돌리기 어려운 행동(외부 호출·삭제·전송) → 전용 툴로 만들어 **승인 게이트**.
  - 읽기 전용(grep/조회) → 병렬 안전 표시.

### 좋은 툴 정의
- 이름·설명을 명확히, **"언제 호출하는지"** 를 설명에 박는다(최신 모델은 보수적으로 호출하므로 트리거 조건이 호출률을 높인다).
- 입력은 JSON 스키마로. 파괴적 작업은 검증/확인.

---

## 4. 구현: Tool Runner vs 수동 루프 (Spring + Anthropic Java SDK)

### Tool Runner (권장 — 루프 자동)
```java
@JsonClassDescription("주문 상태를 조회한다. 사용자가 주문/배송 상태를 물을 때 호출.")
static class GetOrderStatus implements Supplier<String> {
    @JsonPropertyDescription("주문 ID")
    public String orderId;
    @Override public String get() {
        return orderService.statusOf(orderId);   // ← 내 비즈니스 로직(클라이언트 툴)
    }
}

BetaToolRunner runner = client.beta().messages().toolRunner(
    MessageCreateParams.builder()
        .model("claude-opus-4-8")
        .maxTokens(16000L)
        .addTool(GetOrderStatus.class)
        .addUserMessage("내 주문 A123 어디까지 왔어?")
        .build());

for (BetaMessage msg : runner) {   // SDK가 호출→실행→재투입 루프를 대신 돌림
    // 최종/중간 메시지 처리
}
```

### 수동 루프 (정밀 제어: 승인 게이트·로깅·조건 실행)
```java
List<MessageParam> messages = new ArrayList<>();
messages.add(MessageParam.builder().role(USER).content(userInput).build());

while (true) {
    Message res = client.messages().create(params(messages, tools));
    messages.add(res.toParam());                       // assistant 턴 보존(중요)

    if ("end_turn".equals(res.stopReason())) break;

    List<ContentBlockParam> results = new ArrayList<>();
    for (ContentBlock b : res.content()) {
        b.toolUse().ifPresent(tu -> {
            // ⛔ 파괴적 툴이면 여기서 사용자 승인 게이트
            String out = execute(tu.name(), tu.input());   // 내 하니스가 실행
            results.add(toolResult(tu.id(), out));          // tool_use_id 매칭
        });
    }
    messages.add(MessageParam.builder().role(USER).contentOfBlockParams(results).build());
}
```

---

## 5. 컨텍스트 관리 (장기 실행의 핵심)
긴 루프는 컨텍스트가 부풀고 비싸진다.
| 기법 | 언제 | 효과 |
|------|------|------|
| **Compaction(압축)** | 컨텍스트 한계 임박 | 과거를 요약 블록으로 축약(서버측). 응답 content를 그대로 되돌려야 함 |
| **Context editing(편집)** | 오래된 tool_result·thinking이 쌓임 | 임계치 기준으로 잘라냄(요약 X, 삭제) |
| **Memory(메모리)** | 세션을 넘는 상태 보존 | 파일/스토어에 기록, 재시작에도 유지 |
| **요약/RAG로 외부화** | 큰 자료 | 통째로 넣지 말고 [[RAG]]로 필요분만 |

---

## 6. MCP (Model Context Protocol)
- 툴/리소스/프롬프트를 **표준 인터페이스**로 노출하는 프로토콜. (GitHub·Slack·DB 등 외부 통합을 규격화)
- 하니스/모델이 MCP 서버에 붙어 그 서버의 툴을 호출. 자체 툴을 매번 재정의하지 않아도 됨.
- 인증/시크릿은 코드/프롬프트가 아니라 **별도 보관소**에 두고 주입(시크릿 누출 방지).

---

## 7. 비용 제어 & 루프 안전장치

에이전트 루프는 **컨텍스트가 반복마다 선형 증가** → 방치하면 비용 폭주.

```java
// 비용·반복 양쪽 상한 설정
int maxIterations = 20;
int maxTokenBudget = 200_000;  // 전체 세션 토큰 상한
int totalTokensUsed = 0;
int iteration = 0;

while (true) {
    if (++iteration > maxIterations) {
        log.warn("Max iterations reached, forcing stop");
        break;
    }

    Message res = client.messages().create(params(messages, tools));
    totalTokensUsed += res.usage().inputTokens() + res.usage().outputTokens();

    if (totalTokensUsed > maxTokenBudget) {
        log.warn("Token budget exceeded: {}", totalTokensUsed);
        break;
    }

    if ("end_turn".equals(res.stopReason())) break;
    // ... tool execution
}
```

### 발산/무한루프 감지

```java
// 같은 툴+인자를 연속 반복하면 루프로 간주
Map<String, Integer> toolCallCount = new HashMap<>();

b.toolUse().ifPresent(tu -> {
    String key = tu.name() + ":" + tu.input().toString();
    int count = toolCallCount.merge(key, 1, Integer::sum);
    if (count >= 3) {
        throw new AgentLoopDetectedException("반복 툴 호출 감지: " + key);
    }
});
```

---

## 8. 멀티 에이전트 패턴

단일 에이전트가 처리하기 어려운 복잡한 작업을 **여러 에이전트로 분업**.

```
[오케스트레이터-서브에이전트 패턴]

  오케스트레이터 에이전트
    │ 전체 계획 수립
    ├── 서브에이전트 A (웹 검색 전담)
    ├── 서브에이전트 B (코드 실행 전담)
    └── 서브에이전트 C (문서 작성 전담)
    │ 결과 취합 → 최종 응답

장점: 각 에이전트가 최소 권한·특화 툴만 보유
단점: 오케스트레이터 실패 시 전체 실패, 통신 오버헤드
```

```
[병렬 에이전트 패턴]

  작업 A ──► 에이전트 1 ──┐
  작업 B ──► 에이전트 2 ──┼──► 결과 집계
  작업 C ──► 에이전트 3 ──┘

적합: 독립적인 서브태스크를 병렬 처리 (코드 리뷰 여러 파일)
```

### 멀티 에이전트 보안 원칙

```
서브에이전트를 신뢰하지 마라.
  → 오케스트레이터가 검증 없이 서브 에이전트 출력을 그대로 실행하면
    서브 에이전트가 프롬프트 인젝션을 당했을 때 전파됨.
  → 서브 에이전트 출력도 외부 데이터처럼 취급, 파싱 후 검증.
```

---

## 9. 가드레일 & 운영
- **권한/승인**: 파괴적 툴은 `always_ask` → 사용자 확인 후 실행.
- **샌드박싱**: 코드/파일 실행은 격리 환경(컨테이너)에서.
- **타임아웃/재시도**: 외부 호출은 [[Resilience4j]] 패턴(서킷·타임아웃)과 함께.
- **멱등성**: 같은 툴 재호출에도 안전하게.
- **관측성**: 모든 툴 호출/토큰/지연을 로깅·추적 → [[Metrics]] · [[Tracing]] · [[Logging]].
- **프롬프트 인젝션 방어**: 외부 데이터가 시스템 지시를 덮지 못하게 신뢰 경계 분리.
- **루프 상한**: 무한 루프/폭주 방지 (위 섹션 7 참고).

---

## 10. 매니지드 에이전트 (직접 루프를 안 돌리고 싶을 때)
- 제공자(예: Anthropic Managed Agents)가 **에이전트 루프 + 툴 실행 컨테이너**를 대신 운영.
- 영속·버전드 에이전트 설정 + 세션 단위 실행 + 이벤트 스트림.
- 직접 하니스를 운영(컨텍스트·관측·격리)할 여력이 없거나, 세션별 작업공간(파일/레포)이 필요할 때 유리.
- 반대로 **자체 런타임/툴이 필요하면** API + tool use로 직접 루프(위 4장).

## 11. 관련
- [[LLM]] · [[RAG]] · [[Prompt-Engineering]] · [[MCP-Skill]] · [[Resilience4j]] · [[Metrics]] · [[Tracing]] · [[MSA]]
- [[Hermes-Runtime-Internals]] — 루프 가드레일·컨텍스트 압축·서브에이전트의 구체 구현(Hermes)
- [[Agentic-RAG]] — 이 루프를 검색에 적용(Self-RAG·CRAG·반복검색·루프 폭주 한도)
- [[Structured-Data-RAG]] — 라우팅·text-to-SQL 도구 호출·SQL 자기수정 루프의 실행 프레임
