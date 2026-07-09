---
tags:
  - pattern
  - architecture
  - ddd
  - domain
created: 2026-06-15
---

# DDD (Domain-Driven Design)

> [!summary] 한 줄 요약
> **복잡한 비즈니스 도메인을** 소프트웨어 설계의 중심에 두고, 도메인 전문가와 개발자가 **같은 언어(Ubiquitous Language)** 로 모델을 만들어 가는 설계 접근법.

> Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003)에서 정립.

---

## 1. 핵심 사상

- 소프트웨어의 가치는 **도메인(업무 영역)의 복잡성을 잘 다루는 것**에서 나온다.
- 기술이 아니라 **도메인 모델**이 설계의 중심.
- 도메인 전문가와 개발자가 **끊임없는 협업**으로 모델을 정제한다.

DDD는 크게 두 층위로 나뉜다.

| 구분                     | 내용                                                                                  |
| ---------------------- | ----------------------------------------------------------------------------------- |
| **전략적 설계 (Strategic)** | 큰 그림 — Bounded Context, Ubiquitous Language, Context Map                            |
| **전술적 설계 (Tactical)**  | 구현 패턴 — Entity, Value Object, Aggregate, Repository, Domain Service, Domain Event 등 |

---

## 2. 전략적 설계 (Strategic Design)

### 2.1. Ubiquitous Language (보편 언어)
- 도메인 전문가와 개발자가 **공유하는 단일 용어 체계**.
- 코드, 문서, 대화, 테스트 모두 같은 용어를 사용.
- 예: "주문을 *확정(confirm)* 한다" → 코드에도 `order.confirm()` 으로 등장.

### 2.2. Bounded Context (경계가 있는 컨텍스트)
- 하나의 모델이 **일관되게 유효한 경계**.
- 같은 단어라도 컨텍스트마다 의미가 다를 수 있다.
  - 예: "상품(Product)" — *주문* 컨텍스트에서는 가격/재고, *카탈로그* 컨텍스트에서는 설명/이미지.
- **MSA의 서비스 경계를 나누는 핵심 기준**이 된다. → [[MSA]]

### 2.3. Context Map (컨텍스트 맵)
여러 Bounded Context 간의 **관계/통합 방식**을 표현.

| 관계 패턴 | 설명 |
|-----------|------|
| **Shared Kernel** | 두 컨텍스트가 일부 모델을 공유 |
| **Customer/Supplier** | 상류(공급자)–하류(소비자) 관계 |
| **Conformist** | 하류가 상류 모델을 그대로 따름 |
| **Anti-Corruption Layer (ACL)** | 외부/레거시 모델이 내 도메인을 오염시키지 않도록 **변환 계층**을 둠 |
| **Open Host Service / Published Language** | 공개 API·표준 포맷으로 통합 |

### 2.4. Subdomain (하위 도메인)
- **Core Domain**: 경쟁력의 핵심 — 가장 많은 투자를 집중.
- **Supporting Subdomain**: 핵심을 보조 (필요하지만 차별화 요소는 아님).
- **Generic Subdomain**: 범용 (인증, 결제 등 — 외부 솔루션 활용 고려).

---

## 3. 전술적 설계 (Tactical Design)

### 3.1. Entity (엔티티)
- **식별자(ID)로 구분**되는 객체. 속성이 바뀌어도 동일성 유지.
- 예: `User(id=1)` 는 이름이 바뀌어도 같은 사용자.

### 3.2. Value Object (값 객체)
- **속성 값으로만 구분**되는 불변 객체. 식별자 없음.
- 예: `Money(1000, "KRW")`, `Address`, `DateRange`.
- 동등성은 값 비교로 판단, 생성 후 변경 불가(immutable).

### 3.3. Aggregate (애그리거트) ⭐
- **함께 변경되어야 하는 객체들의 묶음** + 일관성 경계.
- **Aggregate Root**: 외부에서 접근 가능한 유일한 진입점.
  - 외부는 루트를 통해서만 내부 객체에 접근.
  - 루트가 **불변식(invariant)** 을 보장.
- 트랜잭션 경계 = 애그리거트 경계 (하나의 트랜잭션 = 하나의 애그리거트 수정 권장).

```
Order (Aggregate Root)
 ├─ OrderLine (내부 엔티티)
 ├─ OrderLine
 └─ ShippingAddress (Value Object)

→ 외부는 Order 를 통해서만 OrderLine 조작
→ "주문 총액 = 라인 합계" 같은 불변식을 Order 가 보장
```

> [!tip] 애그리거트 설계 원칙
> - **작게 설계하라** (큰 애그리거트는 동시성·성능 문제).
> - 애그리거트 간 참조는 **ID로** (객체 직접 참조 X).
> - 한 트랜잭션에서 **하나의 애그리거트만** 변경하고, 나머지는 도메인 이벤트로 동기화.

### 3.4. Repository (리포지토리)
- 애그리거트의 **영속성 추상화**. 컬렉션처럼 다루게 해준다.
- 애그리거트 루트 단위로만 제공. → [[Repository-Pattern]]

### 3.5. Domain Service (도메인 서비스)
- 특정 엔티티/값 객체에 자연스럽게 속하지 않는 **도메인 로직**.
- 예: 환율 변환, 여러 애그리거트에 걸친 정책 계산.
- *애플리케이션 서비스*(유스케이스 조율)와 혼동 주의.

### 3.6. Domain Event (도메인 이벤트)
- 도메인에서 **발생한 의미 있는 사건**. 과거형으로 명명.
- 예: `OrderPlaced`, `PaymentCompleted`.
- 애그리거트 간/컨텍스트 간 느슨한 결합과 동기화에 사용. → [[Event-Sourcing]], [[CQRS]]

### 3.7. Factory (팩토리)
- 복잡한 애그리거트/객체 생성 로직을 캡슐화.

---

## 4. 계층 구조 (Layered / Hexagonal)

```
┌───────────────────────────────────────┐
│ Presentation / Interface (API, UI)     │
├───────────────────────────────────────┤
│ Application (유스케이스 조율, 트랜잭션)  │  ← 얇게 유지
├───────────────────────────────────────┤
│ Domain (Entity, VO, Aggregate, 도메인 로직) │  ← 핵심, 의존성 없음
├───────────────────────────────────────┤
│ Infrastructure (DB, 메시징, 외부 연동)  │
└───────────────────────────────────────┘
```

- **의존성 방향은 항상 Domain 을 향한다** (의존성 역전). → [[Hexagonal-Architecture]]
- Domain 계층은 프레임워크·DB에 의존하지 않는 순수 코드여야 한다.

---

## 5. CQRS / Event Sourcing 과의 관계

- **Aggregate ↔ CQRS의 Write 모델**: 애그리거트가 명령을 처리하고 불변식을 지킨다.
- **Domain Event ↔ Event Sourcing**: 애그리거트의 상태 변화를 이벤트로 표현·저장.
- DDD는 [[CQRS]] · [[Event-Sourcing]] · [[Saga-Pattern]] 와 결합해 복잡한 시스템을 다룬다.

---

## 6. 장점 / 단점

### ✅ 장점
- 비즈니스와 코드의 **간극 축소** (Ubiquitous Language).
- 복잡한 도메인을 **구조적으로 관리** 가능.
- 변경에 강한 모델, 명확한 경계 → MSA 설계의 근거.

### ❌ 단점 / 비용
- **학습 곡선이 가파르다.**
- 도메인 전문가와의 **지속적 협업이 필수** (없으면 효과 반감).
- 단순 CRUD 도메인엔 **과한 투자**.

---

## 7. 언제 쓸 것인가

> [!success] 적합
> - 도메인 복잡도가 높고 비즈니스 규칙이 풍부할 때
> - 장기 운영·지속 변경되는 핵심 시스템(Core Domain)
> - 여러 팀/서비스로 나뉘는 대형 시스템(MSA 경계 설계)

> [!failure] 부적합
> - 단순 CRUD, 데이터 입출력 위주 애플리케이션
> - 도메인 지식 접근이 어렵거나 수명이 짧은 프로젝트

---

## 8. 흔한 오해 / 주의점

- **DDD = 전술 패턴 모음이 아니다.** Entity/Repository만 쓴다고 DDD가 아니며, *전략적 설계(경계·언어)* 가 본질.
- **Anemic Domain Model(빈혈 모델) 안티패턴**: 로직이 서비스에만 있고 엔티티는 getter/setter 덩어리 → 도메인 모델의 의미 상실.
- 모든 곳에 애그리거트를 크게 잡지 말 것 — 작게, 일관성 경계 중심으로.

---

## 10. Spring 구현 예시

> Spring Boot + JPA로 전술 패턴(Entity / Value Object / Aggregate / Repository / Domain Service / Domain Event)을 구현하는 전형적 예.

### 10.1. Value Object — `@Embeddable` + 불변
```java
@Embeddable
public class Money {
    private final long amount;
    @Column(length = 3)
    private final String currency;

    protected Money() { this.amount = 0; this.currency = "KRW"; }  // JPA용

    public Money(long amount, String currency) {
        if (amount < 0) throw new IllegalArgumentException("음수 금액 불가");
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {                 // 변경이 아니라 새 객체 반환(immutable)
        require(this.currency.equals(other.currency));
        return new Money(this.amount + other.amount, this.currency);
    }
    // equals/hashCode는 값 기준 (lombok @Value 또는 record 권장)
}
```

### 10.2. Entity & Aggregate Root — 불변식은 루트가 보장
```java
@Entity
@Table(name = "orders")
public class Order extends AbstractAggregateRoot<Order> {   // 도메인 이벤트 발행 지원
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @ElementCollection                         // 내부 객체는 루트를 통해서만 조작
    @CollectionTable(name = "order_lines", joinColumns = @JoinColumn(name = "order_id"))
    private List<OrderLine> lines = new ArrayList<>();

    protected Order() {}

    // 팩토리 메서드 - 생성 규칙 캡슐화
    public static Order place(CustomerId customerId, List<OrderLine> lines) {
        if (lines.isEmpty()) throw new IllegalArgumentException("주문 항목이 비어 있음");
        Order o = new Order();
        o.status = OrderStatus.PLACED;
        o.lines = new ArrayList<>(lines);
        o.registerEvent(new OrderPlacedEvent(o.id));   // 도메인 이벤트 등록
        return o;
    }

    // 비즈니스 메서드 - 상태 전이 + 불변식 검증을 내부에서
    public void cancel() {
        if (status == OrderStatus.SHIPPED)
            throw new IllegalStateException("이미 배송된 주문은 취소 불가");
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(id));
    }

    public Money totalAmount() {                       // 불변식 계산은 루트가 책임
        return lines.stream().map(OrderLine::subtotal)
                    .reduce(new Money(0, "KRW"), Money::add);
    }
}
```
> ⚠️ **Anemic Domain Model 회피**: `setStatus()` 같은 setter 대신 `cancel()` 처럼 *의도가 담긴 메서드*를 둔다.

### 10.3. Repository — 애그리거트 루트 단위
```java
// 도메인 계층: 인터페이스 (영속성 기술 모름)
public interface OrderRepository {
    Optional<Order> findById(UUID id);
    Order save(Order order);
}

// 인프라 계층: Spring Data 구현
public interface JpaOrderRepository
        extends OrderRepository, JpaRepository<Order, UUID> {}
```

### 10.4. Domain Service — 한 엔티티에 안 맞는 도메인 로직
```java
@Service
@RequiredArgsConstructor
public class ExchangeService {     // 환율 변환은 특정 애그리거트 소유가 아님
    private final ExchangeRateProvider rateProvider;

    public Money convert(Money money, String targetCurrency) {
        BigDecimal rate = rateProvider.getRate(money.currency(), targetCurrency);
        return new Money(/* ... */, targetCurrency);
    }
}
```

### 10.5. Application Service — 유스케이스 조율(얇게) + 트랜잭션
```java
@Service
@RequiredArgsConstructor
public class PlaceOrderUseCase {
    private final OrderRepository orderRepository;

    @Transactional                          // 트랜잭션 경계 = 애그리거트 단위
    public UUID place(PlaceOrderCommand cmd) {
        Order order = Order.place(cmd.customerId(), cmd.toLines());
        return orderRepository.save(order).getId();
    }
}
```

### 10.6. Domain Event 구독 — 애그리거트 간 결합 분리
```java
@Component
public class OrderEventHandler {
    @TransactionalEventListener(phase = AFTER_COMMIT)   // 트랜잭션 커밋 후 처리
    public void on(OrderPlacedEvent event) {
        // 다른 애그리거트/컨텍스트는 이벤트로 동기화 (한 트랜잭션에 하나의 애그리거트)
        log.info("주문 생성됨: {}", event.orderId());
    }
}
```

### 10.7. Anti-Corruption Layer (ACL) — 외부 모델 차단
```java
// 외부(레거시) 응답을 우리 도메인 모델로 변환하는 번역 계층
@Component
@RequiredArgsConstructor
public class LegacyCustomerAdapter implements CustomerProvider {
    private final LegacyCustomerClient legacyClient;     // 외부 SDK/Feign

    @Override
    public Customer findById(CustomerId id) {
        LegacyCustomerResponse res = legacyClient.get(id.value());
        return new Customer(new CustomerId(res.getCustNo()),  // 외부 용어 → 도메인 용어 변환
                            new CustomerName(res.getNm()));
    }
}
```

---

## 9. 관련 패턴 / 개념

- [[CQRS]] — Write 모델로 애그리거트 활용
- [[Event-Sourcing]] — 도메인 이벤트로 상태 저장
- [[Saga-Pattern]] — 여러 애그리거트/서비스에 걸친 트랜잭션
- [[Repository-Pattern]] — 애그리거트 영속성 추상화
- [[Hexagonal-Architecture]] — 도메인 중심 의존성 구조
- [[MSA]] — Bounded Context가 서비스 경계의 기준

---

## 참고
- Eric Evans, *Domain-Driven Design* (2003)
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013)
- Vaughn Vernon, *Domain-Driven Design Distilled* (2016)
