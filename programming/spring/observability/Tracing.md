---
tags:
  - observability
  - tracing
  - opentelemetry
  - distributed-tracing
created: 2026-06-15
---

# Distributed Tracing (분산 추적)

> [!summary] 한 줄 요약
> 하나의 요청이 **여러 서비스를 거치는 전체 경로**를 추적해 어디서 지연/오류가 났는지 파악. Trace(전체) = 여러 Span(구간)의 트리. 표준은 **OpenTelemetry**.

---

## 1. 왜 필요한가
[[MSA]]에서 요청은 Gateway → A → B → DB → C 식으로 흐른다. 단일 로그로는 "어디서 느렸나"를 알 수 없다 → 추적이 서비스 경계를 넘어 흐름을 잇는다.

## 2. 핵심 개념
- **Trace**: 요청 전체 (고유 `traceId`).
- **Span**: 작업 단위 구간 (`spanId`, 부모 span, 시작/종료 시각, 태그).
- **Context Propagation**: 서비스 호출 시 헤더(`traceparent`)로 추적 컨텍스트 전파.

```
Trace(traceId=abc)
 ├─ Span: Gateway        [10ms]
 │   └─ Span: OrderSvc   [120ms]  ← 여기가 느림!
 │       ├─ Span: DB query [90ms]
 │       └─ Span: call PaymentSvc [25ms]
```

## 3. Spring 연동 — Micrometer Tracing + OTel
```groovy
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```
```yaml
management:
  tracing:
    sampling:
      probability: 1.0          # 개발 100%, 운영은 0.1 등으로 샘플링
  otlp:
    tracing:
      endpoint: http://tempo:4318/v1/traces
```
- RestClient/WebClient/Feign/Kafka 등 호출에 **자동 계측** → traceId 전파 + span 생성.
- 로그 MDC에 traceId/spanId 자동 주입 → [[Logging]]과 연계.

### 수동 Span
```java
@Observed(name = "order.process")   // Micrometer Observation → span 생성
public void process(Order o) { ... }
```

## 4. 백엔드/시각화
- **Tempo**(Grafana) / **Jaeger** / **Zipkin** 에 span 저장.
- Grafana에서 trace 뷰 + 로그/메트릭 상관(correlation).

## 5. 베스트 프랙티스
- 운영은 **샘플링**으로 비용 관리(전수 추적은 부담).
- traceId를 로그·에러응답에 노출 → 장애 분석 단축.
- 비동기/메시징([[Kafka]])도 컨텍스트 전파 설정.

## 6. 비동기·에이전트 경계 컨텍스트 전파 ⭐

> [!warning] 비동기 경계에서 trace가 끊기면 분산 디버깅 불가
> trace context(traceId/spanId)는 보통 **ThreadLocal/MDC**에 담긴다. 스레드를 넘는 순간(비동기·메시징·다른 프로세스) **자동으로 따라가지 않아** trace가 끊긴다 — 가장 흔한 관측 실패.

### 6.1 비동기 경계 (@Async / CompletableFuture / 가상스레드)

```
문제: @Async·executor.submit()로 다른 스레드 실행 → ThreadLocal(MDC·trace) 미전파
      → 자식 작업이 새 trace로 찍히거나 traceId 유실

대응:
  - Micrometer의 ContextPropagation + context-propagation 라이브러리
  - TaskDecorator로 MDC/Observation 컨텍스트를 작업 스레드에 복사
  - @Async는 ObservationRegistry 연동된 executor 사용
  - 가상스레드도 동일 — VT라고 자동 전파되지 않음(스레드 경계는 여전히 경계)
```
```java
// Executor에 컨텍스트 전파 데코레이터
@Bean
public TaskDecorator otelTaskDecorator() {
    return runnable -> {
        Map<String,String> ctx = MDC.getCopyOfContextMap();   // 부모 컨텍스트 캡처
        return () -> {
            if (ctx != null) MDC.setContextMap(ctx);
            try { runnable.run(); } finally { MDC.clear(); }
        };
    };
}
```

### 6.2 메시징 경계 (Kafka produce → consume)

```
프로세스가 다름 → 메시지 "헤더"에 trace context를 실어 전파
  Producer: 현재 trace context를 Kafka 헤더(traceparent)에 주입(inject)
  Consumer: 헤더에서 추출(extract)해 새 span을 부모에 연결
  → Spring Kafka + Micrometer Tracing은 observation 활성 시 자동 주입/추출
  (W3C traceparent 헤더 표준)
```
→ 메시지 흐름 추적은 [[Kafka]] 팬아웃·완료 판단과 결합.

### 6.3 LLM 에이전트 멀티홉 (A→B→C)

```
에이전트 간 호출도 "경계" → trace 연결 필요
  - traceId: 기술적 추적(span 트리)
  - correlationId: 비즈니스 워크플로우 식별 (→ [[Multi-Tenant-Memory-Agent]])
  → 둘을 함께 전파: traceparent 헤더 + correlationId를 페이로드/헤더에
  → 한 사용자 요청의 에이전트 체인 전체를 하나의 trace로 시각화
```
→ [[Multi-Agent-Coding-Team]] 디버깅, [[AOP-Logging]] MDC traceId 생성과 연결.

### 6.4 OpenTelemetry Context Propagation

```
Context: trace·baggage를 담는 불변 객체 (ThreadLocal에 현재 컨텍스트 보관)
Propagator: 경계에서 inject(나갈 때)/extract(들어올 때) — W3C traceparent·baggage
Baggage: trace를 따라 흐르는 key-value (예: tenant_id·user_id) — 전 구간 전파
  ⚠️ baggage는 모든 홉에 실려 비용·민감정보 주의 (최소화)
```

## 7. 관련
- [[Logging]] · [[Metrics]] · [[MSA]] · [[Spring-Cloud-Gateway]]
- [[Kafka]] (메시징 전파) · [[AOP-Logging]] (MDC traceId) · [[Multi-Agent-Coding-Team]] (에이전트 추적)
