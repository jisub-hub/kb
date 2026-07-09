---
tags:
  - pattern
  - architecture
  - saga
  - msa
  - distributed-transaction
created: 2026-06-15
---

# Saga Pattern (사가 패턴)

> [!summary] 한 줄 요약
> 여러 서비스에 걸친 **분산 트랜잭션**을, 각 서비스의 **로컬 트랜잭션 + 실패 시 보상 트랜잭션(compensating transaction)** 의 연쇄로 처리하는 패턴.

---

## 1. 왜 필요한가

[[MSA]]에서는 서비스마다 DB가 분리되어 **하나의 ACID 트랜잭션으로 묶을 수 없다**. (2PC는 가용성·성능 문제로 기피.)
→ 대신 **결과적 일관성**을 받아들이고, 단계별 로컬 트랜잭션을 이어 붙인다. 실패하면 앞 단계를 **되돌리는 보상 작업**을 실행.

```
주문 생성 → 결제 → 재고 차감 → 배송 등록
   각 단계는 독립 로컬 트랜잭션
   실패 시 → 역순으로 보상 (결제 취소, 재고 복원 ...)
```

---

## 2. 두 가지 방식

### 2.1. Choreography (코레오그래피, 안무형)
- 중앙 조율자 없음. 각 서비스가 **이벤트를 발행/구독**하며 다음 단계를 트리거.

```
OrderService ─OrderCreated─► PaymentService ─PaymentDone─► InventoryService ...
```

- ✅ 단순, 느슨한 결합, 단일 장애점 없음
- ❌ 흐름이 코드에 흩어져 **전체 파악 어려움**, 순환 의존 위험

### 2.2. Orchestration (오케스트레이션, 지휘형)
- 중앙 **Orchestrator(사가 관리자)** 가 각 단계를 명령으로 호출하고 결과에 따라 다음/보상을 결정.

```
        ┌────────────────┐
        │  Orchestrator  │
        └────────────────┘
        ▼     ▼     ▼
   Payment  Inventory  Shipping
```

- ✅ 흐름이 **한 곳에 명시적**, 복잡한 로직 관리 쉬움
- ❌ Orchestrator에 로직 집중, 추가 컴포넌트 필요

> [!tip]
> 단계가 적고 단순하면 **Choreography**, 흐름이 복잡하고 가시성이 중요하면 **Orchestration**.

---

## 3. 핵심 원칙

- **보상 트랜잭션**은 의미적 되돌리기 (물리적 롤백 X). 예: "결제 취소", "재고 환원".
- 각 단계는 **멱등성(idempotent)** 을 보장해야 함 (재시도 대비).
- **결과적 일관성** — 중간 상태가 외부에 노출될 수 있음(예: PENDING).
- 보상 불가능한 작업(메일 발송 등)은 **마지막에** 배치하거나 별도 처리.

---

## 4. Spring 구현 예시

### 4.1. Orchestration — 사가 오케스트레이터 (동기 호출 버전)
```java
@Service
@RequiredArgsConstructor
public class OrderSagaOrchestrator {
    private final PaymentClient paymentClient;       // 다른 서비스 (Feign 등)
    private final InventoryClient inventoryClient;
    private final ShippingClient shippingClient;

    public void createOrder(OrderContext ctx) {
        try {
            paymentClient.charge(ctx.orderId(), ctx.amount());        // step 1
            try {
                inventoryClient.reserve(ctx.orderId(), ctx.items());  // step 2
                try {
                    shippingClient.schedule(ctx.orderId());           // step 3
                } catch (Exception e) {
                    inventoryClient.release(ctx.orderId(), ctx.items()); // 보상 2
                    paymentClient.refund(ctx.orderId());                 // 보상 1
                    throw e;
                }
            } catch (Exception e) {
                paymentClient.refund(ctx.orderId());                  // 보상 1
                throw e;
            }
        } catch (Exception e) {
            throw new SagaFailedException("주문 사가 실패: " + ctx.orderId(), e);
        }
    }
}
```
> 단계가 많아지면 위 중첩 try/catch 대신 **상태 머신**(Spring StateMachine) 또는 사가 단계 리스트 + 보상 스택으로 일반화한다.

### 4.2. Orchestration — 상태 저장형 + 이벤트 (비동기 버전)
```java
@Entity
public class OrderSagaState {
    @Id private UUID orderId;
    @Enumerated(EnumType.STRING)
    private SagaStep step;        // STARTED, PAYMENT_DONE, STOCK_RESERVED, COMPLETED, COMPENSATING ...
}

@Component
@RequiredArgsConstructor
public class OrderSaga {
    private final SagaStateRepository stateRepo;
    private final StreamBridge streamBridge;     // Spring Cloud Stream (Kafka 등)

    public void start(OrderContext ctx) {
        stateRepo.save(new OrderSagaState(ctx.orderId(), SagaStep.STARTED));
        streamBridge.send("payment-out", new ChargeCommand(ctx.orderId(), ctx.amount()));
    }

    @KafkaListener(topics = "payment-events")
    public void onPayment(PaymentEvent e) {
        if (e.success()) {
            advance(e.orderId(), SagaStep.PAYMENT_DONE);
            streamBridge.send("inventory-out", new ReserveCommand(e.orderId()));
        } else {
            fail(e.orderId());                   // 보상 시작 (앞 단계 없음)
        }
    }

    @KafkaListener(topics = "inventory-events")
    public void onInventory(InventoryEvent e) {
        if (e.success()) advance(e.orderId(), SagaStep.STOCK_RESERVED);
        else {
            streamBridge.send("payment-out", new RefundCommand(e.orderId())); // 보상
            advance(e.orderId(), SagaStep.COMPENSATING);
        }
    }
    // advance/fail: 상태 갱신 헬퍼
}
```

### 4.3. Choreography — 이벤트 발행/구독만 (오케스트레이터 없음)
```java
// 결제 서비스: 결제 완료 시 이벤트 발행
@Service @RequiredArgsConstructor
public class PaymentService {
    private final StreamBridge streamBridge;

    @Transactional
    public void charge(UUID orderId, long amount) {
        // ... 결제 로컬 트랜잭션 ...
        streamBridge.send("payment-events", new PaymentCompleted(orderId));
    }
}

// 재고 서비스: 결제 이벤트를 구독해 다음 단계 진행
@Component @RequiredArgsConstructor
public class InventoryEventHandler {
    private final InventoryService inventoryService;

    @KafkaListener(topics = "payment-events")
    public void on(PaymentCompleted e) {
        inventoryService.reserve(e.orderId());   // 실패 시 보상 이벤트 발행
    }
}
```

> [!warning] 멱등성
> 메시지는 중복 전달될 수 있다. 각 핸들러는 `처리된 메시지 ID 기록` 등으로 **멱등 처리**해야 한다. 이벤트 발행 신뢰성은 [[Outbox-Pattern]] 으로 보장한다.

---

## 5. 보상의 일반화 — 보상 스택 (Compensation Stack)

4.1의 중첩 try/catch는 단계가 늘면 유지 불가다. 실무에서는 **성공한 단계를 스택에 쌓고, 실패 시 역순으로 보상**한다.

```java
public record SagaStep(String name, Runnable action, Runnable compensation) {}

public class SagaExecutor {
    public void run(List<SagaStep> steps) {
        Deque<SagaStep> completed = new ArrayDeque<>();   // 성공한 단계 스택
        try {
            for (SagaStep step : steps) {
                step.action().run();        // 로컬 트랜잭션 실행
                completed.push(step);       // 성공 → 스택에 push
            }
        } catch (Exception e) {
            while (!completed.isEmpty()) {  // 실패 → 역순(LIFO) 보상
                SagaStep s = completed.pop();
                try { s.compensation().run(); }
                catch (Exception ce) { log.error("보상 실패 {} — 수동개입 필요", s.name(), ce); }
            }
            throw new SagaFailedException(e);
        }
    }
}

// 사용
executor.run(List.of(
    new SagaStep("payment",   () -> payment.charge(id),   () -> payment.refund(id)),
    new SagaStep("inventory", () -> inventory.reserve(id),() -> inventory.release(id)),
    new SagaStep("shipping",  () -> shipping.schedule(id),() -> shipping.cancel(id))
));
```

> **보상 실패는 별도 문제다.** 보상마저 실패하면 자동 복구 불가 → DLQ/알림으로 **수동 개입(human-in-the-loop)** 경로를 반드시 둔다.

---

## 6. 타임아웃 · 장애 복구 — 사가의 진짜 난제

분산 환경에선 "응답이 실패"가 아니라 **"응답이 아예 안 온다"**가 더 흔하고 어렵다.

```
문제 1: 단계 응답 무한 대기
  → 각 단계에 타임아웃 설정. 초과 시 "실패"로 간주하고 보상 시작
  → 단, 명령은 도착해 처리됐는데 응답만 유실됐을 수 있음 → 멱등 보상 필수

문제 2: 오케스트레이터 크래시 (비동기 사가)
  → 사가 상태(SagaStep)를 매 전이마다 DB에 영속화 (4.2)
  → 재기동 시 미완료 사가를 스캔해 마지막 상태부터 재개
  → 스케줄러가 "PENDING/COMPENSATING 상태 + 타임아웃 경과" 사가를 주기적으로 복구

문제 3: 중복 명령/이벤트
  → at-least-once 전달이므로 모든 단계·보상은 멱등 (처리된 메시지 ID 기록)
```

```java
// 미완료 사가 복구 스케줄러
@Scheduled(fixedDelay = 30_000)
public void recoverStuckSagas() {
    stateRepo.findStuck(SagaStep.notTerminal(), Instant.now().minusSeconds(60))
        .forEach(saga -> sagaEngine.resumeOrCompensate(saga));  // 재개 또는 보상
}
```

> 타임아웃 값은 **다운스트림 서비스의 p99 응답시간 + 여유**로 잡는다. 너무 짧으면 정상 처리 중인 사가를 보상해버린다.

---

## 7. 격리 부족(ACID의 I 결여) 대응

사가는 ACID의 **Isolation을 포기**한다. 중간 상태가 외부에 노출되어 이상 현상이 생긴다. (Richardson 책의 핵심 주제)

| 이상 현상 | 설명 | 대응책 |
|----------|------|--------|
| **Dirty Read** | 보상 전 중간 상태를 다른 트랜잭션이 읽음 | Semantic Lock (PENDING 상태 플래그) |
| **Lost Update** | 사가 진행 중 다른 트랜잭션이 같은 데이터 덮어씀 | 버전(낙관적 락) / Semantic Lock |
| **Fuzzy Read** | 같은 데이터를 두 번 읽었는데 값이 다름 | 재실행 가능 설계 |

대응 기법:
```
Semantic Lock      — 레코드에 *_PENDING 상태 플래그. 사가 완료/보상까지 다른 작업 차단/대기
Commutative Update — 보상이 순서 무관하도록 설계 (예: 잔액 +100/-100은 교환법칙 성립)
Pessimistic View   — 이상에 취약한 단계를 사가 순서상 뒤로 배치 (보상 영향 최소화)
Reread Value       — 업데이트 전 값을 다시 읽어 변경 여부 확인 (lost update 방지)
By Value           — 요청 위험도에 따라 사가(결과적 일관성) vs 2PC(강한 일관성) 선택
```

> 가장 흔한 실무 해법은 **Semantic Lock**: 주문을 `PAYMENT_PENDING` 같은 중간 상태로 두고, 그 상태에선 다른 작업이 손대지 못하게 한다. 결과적 일관성의 "중간 노출"을 상태로 명시하는 것.

---

## 8. 관찰성 · 완료 추적

사가는 여러 서비스·메시지에 흩어지므로 **추적 가능성이 생명**이다. → [[Kafka]]의 "전체 워크플로우 완료 판단"과 직결.

```
correlationId (=sagaId/orderId)
  → 모든 명령·이벤트·로그에 전파 ([[AOP-Logging]] MDC traceId와 동일 원리)
  → 분산 추적(OpenTelemetry)으로 단계별 소요/실패 지점 시각화

사가 상태 대시보드 (Prometheus):
  saga_started_total / saga_completed_total / saga_compensated_total
  saga_duration_seconds (단계별)
  saga_stuck_count   ← 타임아웃·복구 대상, 알림 연동

완료 판단 지점 (Kafka 8절과 동일):
  Orchestration → 오케스트레이터가 모든 단계 COMPLETED 확인 시
  Choreography  → 마지막 단계 서비스가 OrderCompleted 발행 시
```

---

## 9. 구현 프레임워크

| 도구 | 특징 | 적합 |
|------|------|------|
| 직접 구현 (5절 보상 스택) | 의존성 없음, 단순 사가 | 단계 적고 흐름 단순 |
| **Spring StateMachine** | 상태/전이 명시적 정의 | 중간 복잡도 오케스트레이션 |
| **Temporal / Cadence** | 워크플로우를 코드로, 자동 상태영속·재시도·타임아웃 | 복잡·장기 실행 사가 |
| **Camunda / Zeebe** | BPMN 기반, 비즈니스 가시성 | 비개발자 협업, 감사 |
| **Axon Framework** | Event Sourcing + Saga 통합 | [[Event-Sourcing]]/[[CQRS]] 기반 |
| **Eventuate Tram** | Richardson의 사가 프레임워크 | 교과서적 사가 |

> 단계가 많고 타임아웃·재시도·복구가 중요하면 직접 구현보다 **Temporal** 같은 워크플로우 엔진이 안전하다(6절 문제들을 프레임워크가 처리).

---

## 10. 안티패턴 & 체크리스트

```
❌ 보상 없는 사가              → 실패 시 데이터 불일치 방치
❌ 멱등성 누락                → at-least-once 중복으로 이중 결제/차감
❌ 보상 실패 무시             → 수동개입 경로(DLQ/알림) 없으면 영구 불일치
❌ 타임아웃 없음              → 멈춘 사가가 리소스(Semantic Lock) 영구 점유
❌ 중간 상태 미노출           → PENDING 없이 dirty read 발생
❌ 보상 불가 작업을 앞단에     → 메일발송/외부결제는 사가 마지막에 배치

✅ 체크리스트
  □ 모든 단계에 보상 트랜잭션 정의 (의미적 되돌리기)
  □ 모든 단계·보상 멱등
  □ 각 단계 타임아웃 + 미완료 사가 복구 스케줄러
  □ 사가 상태 영속화 (오케스트레이터 크래시 복구)
  □ Semantic Lock으로 중간 상태 격리
  □ correlationId 전파 + 사가 상태 모니터링/알림
  □ 보상 실패 시 human-in-the-loop 경로
  □ 이벤트 발행 원자성 → [[Outbox-Pattern]]
```

---

## 11. 관련 패턴

- [[MSA]] — 분산 트랜잭션 문제의 배경
- [[Kafka]] — 사가 이벤트 버스, 완료 판단(8절)·Aggregator
- [[Event-Sourcing]] · [[CQRS]] — 이벤트 기반 사가 구현
- [[Outbox-Pattern]] — 이벤트 발행의 원자성 보장
- [[DDD]] — 애그리거트 = 로컬 트랜잭션 단위
- [[Transaction-Management]] — 사가의 각 단계인 로컬 트랜잭션(@Transactional 전파·경계)

## 참고
- Chris Richardson, *Microservices Patterns* — Saga
- microservices.io/patterns/data/saga.html
