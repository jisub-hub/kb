---
tags:
  - pattern
  - architecture
  - msa
  - microservices
created: 2026-06-15
---

# MSA (Microservices Architecture)

> [!summary] 한 줄 요약
> 하나의 애플리케이션을 **독립적으로 배포 가능한 작은 서비스들**로 나누고, 각 서비스가 **자체 데이터·비즈니스 역량**을 갖고 가볍게 통신하는 아키텍처.

---

## 1. 개념 & 특징

- 서비스는 **하나의 비즈니스 역량(Bounded Context)** 중심으로 분리 → [[DDD]].
- 각 서비스는 **독립 배포·독립 확장·독립 기술 스택** 가능.
- **Database per Service**: 서비스마다 DB 소유 (공유 DB 금지).
- 통신: 동기(REST/gRPC) + 비동기(메시지/이벤트).

```
[Monolith]                 [MSA]
┌──────────────┐     ┌─────┐ ┌─────┐ ┌─────┐
│  하나의 앱     │ →  │주문 │ │결제 │ │배송 │
│ (단일 배포)   │     └──┬──┘ └──┬──┘ └──┬──┘
└──────────────┘       DB     DB     DB
```

---

## 2. 장점 / 단점

### ✅ 장점
- 서비스별 **독립 배포/확장**, 장애 격리.
- 팀별 **자율성**(기술·릴리즈 주기).
- 부분적 기술 교체 용이.

### ❌ 단점
- **분산 시스템의 복잡성**: 네트워크, 지연, 부분 실패.
- **분산 트랜잭션** 불가 → [[Saga-Pattern]], 결과적 일관성.
- 운영/모니터링/배포 자동화 부담 (CI/CD, 관측성 필수).
- 잘못 나누면 **분산 모놀리스** (최악).

---

## 3. 관련 핵심 패턴

| 영역 | 패턴 |
|------|------|
| 서비스 경계 | [[DDD]] Bounded Context |
| 데이터 일관성 | [[Saga-Pattern]], [[CQRS]], [[Event-Sourcing]] |
| 신뢰성 메시징 | [[Outbox-Pattern]] |
| 통신/진입 | API Gateway, BFF |
| 회복탄력성 | Circuit Breaker, Retry, Bulkhead |
| 관측성 | 분산 추적, 중앙 로깅, 메트릭 |

---

## 4. 언제 쓸 것인가

> [!success] 적합
> - 큰 조직·여러 팀, 서비스별 독립 배포·확장 필요
> - 도메인이 명확히 분리되고 복잡도가 높을 때

> [!failure] 부적합 / 주의
> - 초기 스타트업·소규모 팀 → **모놀리스로 시작** 후 필요 시 분리 권장
> - 도메인 경계가 불명확하면 분산 모놀리스 위험

> [!tip] Monolith First
> 대부분의 경우 잘 만든 모듈러 모놀리스로 시작하고, 경계가 검증된 뒤 MSA로 추출하는 것이 안전하다.

---

## 6. Spring 구현 예시 (Spring Cloud)

### 6.1. 서비스 간 통신 — OpenFeign (선언적 HTTP 클라이언트)
```java
@FeignClient(name = "payment-service")        // 서비스 이름으로 호출 (디스커버리 연동)
public interface PaymentClient {
    @PostMapping("/payments")
    PaymentResult charge(@RequestBody ChargeRequest request);
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentClient paymentClient;   // REST 호출이 메서드처럼 추상화됨

    public void placeOrder(OrderCommand cmd) {
        PaymentResult result = paymentClient.charge(new ChargeRequest(cmd.amount()));
        // ...
    }
}
```

### 6.2. 회복탄력성 — Resilience4j Circuit Breaker
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentClient paymentClient;

    @CircuitBreaker(name = "payment", fallbackMethod = "fallback")  // 장애 격리
    @Retry(name = "payment")
    public PaymentResult charge(ChargeRequest req) {
        return paymentClient.charge(req);
    }

    // 회로가 열렸거나 실패 시 대체 응답 (장애 전파 차단)
    private PaymentResult fallback(ChargeRequest req, Throwable t) {
        return PaymentResult.pending();   // 또는 큐 적재 후 재시도
    }
}
```
```yaml
resilience4j.circuitbreaker.instances.payment:
  failure-rate-threshold: 50          # 실패율 50% 초과 시 회로 open
  wait-duration-in-open-state: 10s
  sliding-window-size: 20
```

### 6.3. API Gateway — Spring Cloud Gateway
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service        # 로드밸런싱 + 디스커버리
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args: { name: orderCB, fallbackUri: forward:/fallback/order }
```

### 6.4. 비동기 이벤트 — Spring Cloud Stream
```java
// 발행
@Service @RequiredArgsConstructor
public class OrderService {
    private final StreamBridge streamBridge;
    public void completed(UUID orderId) {
        streamBridge.send("order-events", new OrderCompleted(orderId));
    }
}

// 구독 (함수형 바인딩)
@Bean
public Consumer<OrderCompleted> handleOrderCompleted() {
    return event -> log.info("주문 완료 수신: {}", event.orderId());
}
```

> [!tip]
> 분산 트랜잭션은 [[Saga-Pattern]], 이벤트 발행 신뢰성은 [[Outbox-Pattern]], 서비스 경계 설계는 [[DDD]] Bounded Context를 함께 보라. 관측성(분산 추적)은 **Micrometer Tracing + OpenTelemetry**로 구성한다.

---

## 5. 관련 패턴

- [[DDD]] · [[CQRS]] · [[Event-Sourcing]] · [[Saga-Pattern]] · [[Outbox-Pattern]] · [[Hexagonal-Architecture]]

## 참고
- Sam Newman, *Building Microservices*
- Chris Richardson, *Microservices Patterns*
- microservices.io
