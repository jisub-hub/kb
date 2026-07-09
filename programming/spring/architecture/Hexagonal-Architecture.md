---
tags:
  - pattern
  - architecture
  - hexagonal
  - ports-and-adapters
  - clean-architecture
created: 2026-06-15
---

# Hexagonal Architecture (헥사고날 / Ports & Adapters)

> [!summary] 한 줄 요약
> 애플리케이션 **핵심(도메인)** 을 외부(DB·UI·메시징)로부터 격리하고, 모든 입출력을 **Port(인터페이스)** 와 **Adapter(구현)** 를 통해 연결하는 아키텍처. (Alistair Cockburn)

---

## 1. 개념

```
        ┌─────── Driving(주도) Adapters ───────┐
        │  REST Controller, CLI, Test          │
        └──────────────┬───────────────────────┘
                       ▼ (Inbound Port)
              ┌───────────────────┐
              │   Application /    │
              │   Domain Core      │  ← 외부에 의존하지 않음
              └───────────────────┘
                       ▼ (Outbound Port)
        ┌──────────────┴───────────────────────┐
        │  DB Adapter, MQ Adapter, 외부 API     │
        └─────── Driven(피동) Adapters ─────────┘
```

- **Port**: 핵심이 외부와 소통하는 **인터페이스** (핵심이 소유).
  - Inbound(Driving) Port: 외부 → 핵심 호출 (유스케이스).
  - Outbound(Driven) Port: 핵심 → 외부 호출 (Repository, 메시지 발행 등).
- **Adapter**: Port의 **구체 구현** (프레임워크/기술 종속 코드).

---

## 2. 핵심 원칙 — 의존성 역전

- **의존성은 항상 핵심(도메인)을 향한다.**
- 도메인은 DB·웹·프레임워크를 **모른다**. 인터페이스(Port)만 안다.
- 기술 교체(예: MySQL → MongoDB, REST → gRPC)는 **Adapter만 교체**.

> Clean Architecture / Onion Architecture 도 본질적으로 같은 사상(의존성 역전 + 도메인 중심).

---

## 3. 장점 / 단점

### ✅ 장점
- 도메인 로직의 **높은 테스트 용이성** (외부를 목으로 대체).
- 기술 스택 교체에 유연.
- 관심사 분리 명확. [[DDD]]와 궁합이 좋음.

### ❌ 단점
- 인터페이스·계층이 늘어 **보일러플레이트 증가**.
- 단순 애플리케이션엔 과한 구조.

---

## 5. Spring 구현 예시

> 패키지를 `domain / application / adapter(in, out)` 로 나누고, 의존성이 항상 안쪽(domain)을 향하게 한다.

### 5.1. 패키지 구조
```
com.example.order
 ├─ domain               (순수 도메인, 프레임워크 의존 X)
 │   └─ Order.java
 ├─ application
 │   ├─ port.in
 │   │   └─ PlaceOrderUseCase.java     (Inbound Port)
 │   ├─ port.out
 │   │   └─ LoadOrderPort.java         (Outbound Port)
 │   └─ service
 │       └─ PlaceOrderService.java     (UseCase 구현)
 └─ adapter
     ├─ in.web
     │   └─ OrderController.java        (Driving Adapter)
     └─ out.persistence
         └─ OrderPersistenceAdapter.java (Driven Adapter)
```

### 5.2. Port (인터페이스) — application 계층이 소유
```java
// Inbound Port: 외부 → 핵심 (유스케이스)
public interface PlaceOrderUseCase {
    UUID place(PlaceOrderCommand command);
}

// Outbound Port: 핵심 → 외부 (저장 등). 핵심이 "필요한 것"을 인터페이스로 선언
public interface SaveOrderPort {
    Order save(Order order);
}
```

### 5.3. UseCase 구현 — Port에만 의존
```java
@Service
@RequiredArgsConstructor
public class PlaceOrderService implements PlaceOrderUseCase {
    private final SaveOrderPort saveOrderPort;     // 구현(JPA/Mongo)을 모름

    @Override
    @Transactional
    public UUID place(PlaceOrderCommand command) {
        Order order = Order.place(command.customerId(), command.toLines());
        return saveOrderPort.save(order).getId();
    }
}
```

### 5.4. Driving Adapter — REST Controller (Inbound)
```java
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase;   // Inbound Port에 의존

    @PostMapping
    public ResponseEntity<Void> place(@RequestBody PlaceOrderRequest req) {
        UUID id = placeOrderUseCase.place(req.toCommand());
        return ResponseEntity.created(URI.create("/orders/" + id)).build();
    }
}
```

### 5.5. Driven Adapter — JPA 영속성 (Outbound)
```java
@Component
@RequiredArgsConstructor
public class OrderPersistenceAdapter implements SaveOrderPort {  // Outbound Port 구현
    private final OrderJpaRepository jpaRepository;              // Spring Data

    @Override
    public Order save(Order order) {
        OrderJpaEntity saved = jpaRepository.save(OrderJpaEntity.fromDomain(order));
        return saved.toDomain();                                // 엔티티 ↔ 도메인 매핑
    }
}
```

> [!tip] 핵심 효과
> DB를 MySQL→MongoDB로 바꿔도 `domain`/`application`은 **한 줄도 안 바뀌고**, `adapter.out.persistence`만 교체된다. 테스트 시에도 Outbound Port를 목으로 대체해 도메인 로직만 빠르게 검증할 수 있다.

---

## 4. 관련 패턴

- [[DDD]] — 도메인 중심 설계의 구조적 토대
- [[Repository-Pattern]] — 대표적인 Outbound Port/Adapter
- [[CQRS]] — 유스케이스(Inbound Port)를 Command/Query로 분리

## 참고
- Alistair Cockburn, *Hexagonal Architecture*
- Robert C. Martin, *Clean Architecture*
