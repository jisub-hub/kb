---
tags:
  - pattern
  - architecture
  - cqrs
  - ddd
created: 2026-06-15
---

# CQRS (Command Query Responsibility Segregation)

> [!summary] 한 줄 요약
> **상태를 변경하는 작업(Command)** 과 **상태를 조회하는 작업(Query)** 의 책임과 모델을 분리하는 아키텍처 패턴.

---

## 1. 개념

CQRS는 **명령(Command)** 과 **조회(Query)** 를 서로 다른 모델로 분리한다는 단순한 원칙에서 출발한다.

- **Command**: 시스템의 상태를 **변경**한다. (Create / Update / Delete)
  - 결과로 데이터를 반환하지 않는 것이 원칙 (반환하더라도 ID 정도)
  - "무엇을 하라"는 의도(intent)를 표현 → 예: `PlaceOrderCommand`, `CancelReservationCommand`
- **Query**: 시스템의 상태를 **조회**한다. (Read)
  - 부수효과(side-effect)가 없어야 함 → 순수 조회
  - 화면/응답에 최적화된 형태(DTO, ViewModel)를 반환

> CQRS의 뿌리는 Bertrand Meyer의 **CQS(Command Query Separation)** 원칙이다.
> CQS는 "메서드 단위"의 원칙이고, CQRS는 이를 "객체/모델/아키텍처 단위"로 확장한 것.

```
[CQS]  하나의 객체 안에서 메서드를 command/query로 구분
[CQRS] 모델 자체를 write 모델 / read 모델로 분리
```

---

## 2. 왜 분리하는가 (동기)

전통적인 CRUD 모델은 하나의 모델(엔티티)로 읽기와 쓰기를 모두 처리한다. 단순한 도메인에서는 충분하지만, 복잡해지면 다음 문제가 생긴다.

| 문제 | 설명 |
|------|------|
| **읽기/쓰기 요구 불균형** | 대부분 시스템은 읽기가 쓰기보다 압도적으로 많다. 하나의 모델로는 각각 최적화가 어렵다. |
| **모델 비대화** | 하나의 엔티티가 검증, 조회, 표시 로직을 모두 떠안아 복잡해진다. |
| **조회 성능** | 정규화된 쓰기 스키마는 복잡한 조회(조인 다수)에 불리하다. |
| **권한/검증 혼재** | 쓰기에만 필요한 불변식 검증이 조회 경로에도 끼어든다. |

CQRS는 읽기와 쓰기를 **독립적으로 모델링·확장·최적화**할 수 있게 해준다.

---

## 3. 구조

```
                 ┌─────────────────────────────┐
   Command  ───► │  Command Handler (Write)    │ ───► Write Model / DB
 (상태 변경)      │  - 도메인 로직, 불변식 검증   │        (정규화)
                 └─────────────────────────────┘
                              │
                              │ (이벤트/동기화)
                              ▼
                 ┌─────────────────────────────┐
   Query    ───► │  Query Handler (Read)       │ ◄─── Read Model / DB
 (상태 조회)      │  - 조회 전용, DTO 반환        │        (비정규화/뷰)
                 └─────────────────────────────┘
```

### 핵심 컴포넌트
- **Command / Command Handler**: 명령을 받아 도메인 모델을 변경하고 영속화
- **Query / Query Handler**: 조회 요청을 받아 읽기 모델에서 데이터를 가져와 반환
- **Write Model**: 비즈니스 규칙·불변식 중심 (DDD의 Aggregate와 잘 맞음)
- **Read Model**: 화면/응답에 최적화된 비정규화 뷰

---

## 4. 분리 수준 (단계별)

CQRS는 "전부 아니면 전무"가 아니다. 점진적으로 적용할 수 있다.

### 4.1. 단일 DB, 모델만 분리
- 같은 데이터베이스를 쓰되, **코드 레벨에서 Command/Query 모델만 분리**
- 가장 가볍고 흔한 적용 형태
- 예: 쓰기는 도메인 엔티티/Repository, 읽기는 전용 조회 서비스 + DTO(또는 DB View, Read-only 프로젝션)

### 4.2. 단일 DB, 읽기 전용 복제본 분리
- 쓰기는 Primary, 읽기는 Read Replica로 분산
- 조회 부하를 물리적으로 분리

### 4.3. 별도의 Read/Write 저장소 (완전 분리)
- 쓰기 DB(정규화)와 읽기 DB(비정규화/검색엔진/캐시)를 **물리적으로 분리**
- 두 저장소 간 동기화 필요 → 보통 **이벤트 기반**
- **결과적 일관성(Eventual Consistency)** 을 수용해야 함

> [!tip]
> 처음부터 4.3까지 가지 말 것. **4.1에서 시작**해서 필요할 때 단계를 올리는 것이 현실적이다.

---

## 5. Event Sourcing 과의 관계

CQRS와 **Event Sourcing(ES)** 은 자주 함께 언급되지만 **별개의 패턴**이다.

- **Event Sourcing**: 상태를 직접 저장하는 대신, 상태 변경을 일으킨 **이벤트의 연속**을 저장하고, 현재 상태는 이벤트를 재생(replay)하여 얻는다.
- CQRS는 ES 없이도 가능하고, ES도 CQRS 없이 가능하다.
- 다만 **궁합이 매우 좋다**:
  - Write 측은 이벤트를 발생/저장
  - 그 이벤트로 Read 측의 비정규화된 **Projection(읽기 모델)** 을 갱신

```
Command ─► Aggregate ─► Event 저장 ─► (이벤트 발행)
                                          │
                                          ▼
                                  Projection 갱신 ─► Read Model
```

관련: [[Event-Sourcing]] · [[DDD]]

---

## 6. 장점 / 단점

### ✅ 장점
- **독립적 최적화**: 읽기/쓰기를 각각 스키마·기술·스케일링 측면에서 최적화
- **독립적 확장**: 읽기 부하와 쓰기 부하를 따로 스케일 아웃
- **관심사 분리**: 모델이 단순해지고 도메인 로직이 명확해짐
- **보안**: 쓰기 권한과 읽기 권한을 명확히 분리
- **유연한 읽기 모델**: 화면별로 최적화된 여러 read model 운용 가능

### ❌ 단점 / 비용
- **복잡도 증가**: 모델·인프라가 늘어남 (특히 완전 분리 시)
- **결과적 일관성**: 별도 저장소 동기화 시 읽기가 즉시 최신이 아닐 수 있음 → UX/설계 고려 필요
- **코드 중복**: Command/Query 양쪽에 유사한 구조 발생 가능
- **오버엔지니어링 위험**: 단순 CRUD 도메인엔 과함

---

## 7. 언제 쓰고, 언제 쓰지 말 것인가

> [!success] 적합한 경우
> - 읽기/쓰기 부하나 모델 형태가 **크게 다른** 경우
> - 복잡한 비즈니스 로직 + 복잡한 조회 요구가 **공존**할 때
> - 협업 도메인(동시성·충돌이 잦은 곳), 태스크 기반 UI
> - 높은 확장성/성능이 요구되는 핫스팟 영역
> - DDD, 이벤트 기반 아키텍처, MSA와 결합

> [!failure] 부적합한 경우
> - 단순 CRUD가 대부분인 도메인
> - 팀 규모가 작고 도메인이 단순한 초기 단계
> - 결과적 일관성을 감당할 여력/필요가 없는 경우
>
> → 이런 경우 전통적 CRUD가 더 빠르고 유지보수도 쉽다.

> [!warning]
> "마이크로서비스 = CQRS 필수"는 오해다. **전체 시스템이 아니라 필요한 Bounded Context에만** 선택적으로 적용하라.

---

## 8. 예시 코드 (개념 수준)

### Command 측
```java
// Command: 의도를 표현하는 불변 객체
public record PlaceOrderCommand(String customerId, List<OrderItem> items) {}

@Component
public class PlaceOrderHandler {
    private final OrderRepository orderRepository;
    private final DomainEventPublisher publisher;

    public OrderId handle(PlaceOrderCommand cmd) {
        // 도메인 불변식 검증은 Aggregate 안에서
        Order order = Order.place(cmd.customerId(), cmd.items());
        orderRepository.save(order);
        publisher.publish(new OrderPlacedEvent(order.getId()));  // read model 갱신 트리거
        return order.getId();   // 결과는 식별자 정도만
    }
}
```

### Query 측
```java
// Query: 조회 전용, 화면 최적화 DTO 반환
public record OrderSummaryQuery(String customerId) {}

@Component
public class OrderSummaryHandler {
    private final OrderReadDao readDao;   // 비정규화된 read model 조회

    public List<OrderSummaryDto> handle(OrderSummaryQuery q) {
        // 도메인 엔티티가 아니라 화면용 DTO를 바로 반환
        return readDao.findSummariesByCustomer(q.customerId());
    }
}
```

---

## 9. 흔한 안티패턴 / 주의점

- **Command가 데이터를 잔뜩 반환**: Command는 상태 변경이 목적. 조회까지 떠맡기지 말 것.
- **Read 모델을 도메인 엔티티로 재사용**: read는 DTO/뷰로 가볍게. 무리하게 엔티티 재사용 X.
- **무조건 두 DB로 분리**: 동기화 복잡도와 일관성 비용을 과소평가하지 말 것. 4.1부터 시작.
- **결과적 일관성 미고려**: UI에서 "방금 만든 데이터가 목록에 안 보임" 문제 → 낙관적 갱신, 폴링, 알림 등으로 대응.

---

## 11. Spring 구현 예시

> 핵심: **Command 측(쓰기)** 과 **Query 측(읽기)** 의 패키지·모델·핸들러를 분리한다. 단일 DB로 시작(4.1)하는 가장 흔한 형태.

### 11.1. 패키지 구조
```
com.example.order
 ├─ command
 │   ├─ PlaceOrderCommand.java
 │   ├─ PlaceOrderHandler.java
 │   ├─ Order.java                (Aggregate, @Entity)
 │   └─ OrderRepository.java      (쓰기 전용, Spring Data)
 ├─ query
 │   ├─ OrderSummaryDto.java
 │   ├─ OrderQueryService.java    (읽기 전용)
 │   └─ OrderQueryRepository.java (조회 전용, 프로젝션)
 └─ api
     └─ OrderController.java
```

### 11.2. Command 측 (쓰기)
```java
// 1) Command - 불변 객체(record)로 의도 표현
public record PlaceOrderCommand(String customerId, List<OrderItemDto> items) {}

// 2) Aggregate (쓰기 모델) - Spring Data JPA + 도메인 이벤트
@Entity
@Table(name = "orders")
public class Order extends AbstractAggregateRoot<Order> {   // Spring Data: 도메인 이벤트 지원
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    private String customerId;
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    @ElementCollection
    private List<OrderLine> lines = new ArrayList<>();

    protected Order() {}

    public static Order place(String customerId, List<OrderLine> lines) {
        Order o = new Order();
        o.customerId = customerId;
        o.lines = lines;
        o.status = OrderStatus.PLACED;
        o.registerEvent(new OrderPlacedEvent(o.id, customerId)); // 커밋 후 발행됨
        return o;
    }
}

// 3) 쓰기 전용 Repository
public interface OrderRepository extends JpaRepository<Order, UUID> {}

// 4) Command Handler
@Service
@RequiredArgsConstructor
public class PlaceOrderHandler {
    private final OrderRepository orderRepository;

    @Transactional
    public UUID handle(PlaceOrderCommand cmd) {
        Order order = Order.place(cmd.customerId(), toLines(cmd.items()));
        orderRepository.save(order);     // save 시 registerEvent 한 이벤트가 발행됨
        return order.getId();            // 결과는 식별자 정도만 반환
    }
}
```

### 11.3. Query 측 (읽기)
```java
// 화면 최적화 DTO - 엔티티가 아님
public record OrderSummaryDto(UUID orderId, String customerId,
                              String status, long totalAmount) {}

// 조회 전용 Repository - JPQL/Native 프로젝션 (읽기 모델에 최적화)
public interface OrderQueryRepository extends Repository<Order, UUID> {

    @Query("""
        select new com.example.order.query.OrderSummaryDto(
            o.id, o.customerId, o.status, sum(l.price * l.qty))
        from Order o join o.lines l
        where o.customerId = :customerId
        group by o.id, o.customerId, o.status
    """)
    List<OrderSummaryDto> findSummariesByCustomer(String customerId);
}

@Service
@RequiredArgsConstructor
public class OrderQueryService {
    private final OrderQueryRepository queryRepository;

    @Transactional(readOnly = true)   // 읽기 전용 트랜잭션
    public List<OrderSummaryDto> getSummaries(String customerId) {
        return queryRepository.findSummariesByCustomer(customerId);
    }
}
```

### 11.4. Controller — Command/Query 분리 노출
```java
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {
    private final PlaceOrderHandler placeOrderHandler;   // 쓰기
    private final OrderQueryService orderQueryService;   // 읽기

    @PostMapping                                          // Command
    public ResponseEntity<Void> place(@RequestBody PlaceOrderCommand cmd) {
        UUID id = placeOrderHandler.handle(cmd);
        return ResponseEntity.created(URI.create("/orders/" + id)).build();
    }

    @GetMapping("/summaries")                             // Query
    public List<OrderSummaryDto> summaries(@RequestParam String customerId) {
        return orderQueryService.getSummaries(customerId);
    }
}
```

### 11.5. 읽기 모델을 별도 갱신 (4.3 완전 분리 시)
이벤트로 비정규화 read model 테이블을 갱신:
```java
@Component
@RequiredArgsConstructor
public class OrderSummaryProjection {
    private final OrderSummaryViewRepository viewRepository;

    @TransactionalEventListener(phase = AFTER_COMMIT)  // 쓰기 커밋 후
    public void on(OrderPlacedEvent e) {
        viewRepository.save(new OrderSummaryView(e.orderId(), e.customerId(), "PLACED", 0));
    }
}
```
> 별도 저장소로 분리하면 read model은 **결과적 일관성**을 가진다. (Axon Framework를 쓰면 CommandBus/QueryBus/EventBus가 기본 제공된다.)

---

## 10. 관련 패턴 / 개념

- [[DDD]] — Aggregate가 Write 모델과 자연스럽게 맞음
- [[Event-Sourcing]] — CQRS와 시너지가 큰 별개 패턴
- [[Mediator-Pattern]] — Command/Query 디스패칭에 자주 사용 (예: MediatR)
- [[Eventual-Consistency]] — 저장소 분리 시 핵심 트레이드오프
- [[MSA]] — Bounded Context별 선택 적용

---

## 참고
- Martin Fowler, *CQRS* — https://martinfowler.com/bliki/CQRS.html
- Greg Young — CQRS / Event Sourcing 원전 강연
- Microsoft, *CQRS pattern* (Azure Architecture Center)
