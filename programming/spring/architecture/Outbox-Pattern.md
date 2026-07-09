---
tags:
  - pattern
  - architecture
  - outbox
  - messaging
  - msa
created: 2026-06-15
---

# Transactional Outbox Pattern (트랜잭셔널 아웃박스)

> [!summary] 한 줄 요약
> DB 상태 변경과 메시지 발행을 **하나의 로컬 트랜잭션으로 원자적으로** 처리하기 위해, 메시지를 같은 DB의 **outbox 테이블에 함께 저장**한 뒤 별도 프로세스가 이를 읽어 메시지 브로커로 발행하는 패턴.

---

## 1. 어떤 문제를 푸는가 — Dual Write 문제

```
[문제] 비즈니스 데이터는 DB에, 이벤트는 Kafka에 따로 쓰면?
   1) DB 커밋 성공 → 2) Kafka 발행 실패  ⇒ 불일치!
   (DB와 브로커를 묶는 분산 트랜잭션은 비싸고 취약)
```

→ **둘을 같은 DB 트랜잭션 안에** 넣으면 원자성 확보.

---

## 2. 동작

```
① 비즈니스 트랜잭션
   ├─ orders 테이블 UPDATE
   └─ outbox 테이블 INSERT (이벤트 레코드)   ← 같은 로컬 트랜잭션, 원자적 커밋

② Message Relay (별도 프로세스)
   - outbox 테이블 polling 또는 CDC(예: Debezium, DB 트랜잭션 로그)
   - 미발행 레코드를 메시지 브로커(Kafka 등)로 발행
   - 발행 완료 표시 / 삭제
```

- **CDC(Change Data Capture)** 방식: DB의 트랜잭션 로그를 읽어 발행 → 폴링 부하 없음(예: Debezium).
- 소비자는 **멱등 처리** 필요 (at-least-once 전달이므로 중복 가능).

---

## 3. 장점 / 단점

### ✅ 장점
- DB 변경과 이벤트 발행의 **원자성** 보장 (dual write 문제 해소).
- 분산 트랜잭션(2PC) 없이 신뢰성 있는 메시징.
- [[Saga-Pattern]] · [[Event-Sourcing]] 의 신뢰성 기반.

### ❌ 단점
- **at-least-once** → 중복 전달 가능 → 소비자 멱등성 필수.
- relay/polling 또는 CDC 인프라 추가.
- 발행 순서 보장에 별도 설계 필요할 수 있음.

---

## 5. Spring 구현 예시

### 5.1. Outbox 엔티티
```java
@Entity
@Table(name = "outbox")
public class OutboxMessage {
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    private String aggregateType;     // "Order"
    private String aggregateId;
    private String eventType;         // "OrderPlaced"
    @Column(columnDefinition = "jsonb")
    private String payload;
    private Instant createdAt = Instant.now();
    private boolean published = false;

    public static OutboxMessage of(String type, String aggId, String payload) {
        OutboxMessage m = new OutboxMessage();
        m.aggregateType = "Order"; m.eventType = type;
        m.aggregateId = aggId; m.payload = payload;
        return m;
    }
}

public interface OutboxRepository extends JpaRepository<OutboxMessage, UUID> {
    List<OutboxMessage> findTop100ByPublishedFalseOrderByCreatedAtAsc();
}
```

### 5.2. 비즈니스 변경 + outbox INSERT 를 한 트랜잭션으로
```java
@Service
@RequiredArgsConstructor
public class PlaceOrderService {
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper mapper;

    @Transactional                       // ⭐ 둘이 같은 로컬 트랜잭션 → 원자적 커밋
    public UUID place(PlaceOrderCommand cmd) throws JsonProcessingException {
        Order order = Order.place(cmd.customerId(), cmd.toLines());
        orderRepository.save(order);                          // (1) 비즈니스 데이터

        var event = new OrderPlacedEvent(order.getId(), order.getCustomerId());
        outboxRepository.save(OutboxMessage.of(              // (2) 이벤트도 같은 DB에
            "OrderPlaced", order.getId().toString(), mapper.writeValueAsString(event)));
        return order.getId();
    }
}
```

### 5.3. Message Relay — 폴링 방식
```java
@Component
@RequiredArgsConstructor
public class OutboxRelay {
    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 1000)        // 1초마다 미발행 메시지 발행
    @Transactional
    public void publishPending() {
        var messages = outboxRepository.findTop100ByPublishedFalseOrderByCreatedAtAsc();
        for (OutboxMessage m : messages) {
            kafka.send("order-events", m.getAggregateId(), m.getPayload());
            m.markPublished();           // 발행 표시 (또는 삭제)
        }
        // at-least-once: 발행 후 표시 전에 죽으면 재발행될 수 있음 → 소비자 멱등성 필수
    }
}
```

### 5.4. CDC 방식 (권장, 폴링 부하 없음)
```yaml
# Debezium이 outbox 테이블의 트랜잭션 로그(WAL/binlog)를 읽어 Kafka로 자동 발행.
# 애플리케이션은 outbox INSERT 만 하면 됨 — 별도 relay 코드 불필요.
# Debezium Outbox Event Router 커넥터 설정 예:
transforms: outbox
transforms.outbox.type: io.debezium.transforms.outbox.EventRouter
transforms.outbox.table.field.event.payload: payload
route.by.field: aggregateType        # aggregateType별 토픽 라우팅
```

> [!warning] 소비자 멱등성
> Outbox는 **at-least-once** 전달이다. 소비 측은 `이미 처리한 eventId` 를 기록하거나 upsert로 **중복을 흡수**해야 한다.

---

## 4. 관련 패턴

- [[Saga-Pattern]] — 사가 단계 이벤트 발행 신뢰성
- [[Event-Sourcing]] — 이벤트 발행 보장
- [[CQRS]] — read model 갱신 이벤트 전달
- [[MSA]] — 서비스 간 비동기 통신

## 참고
- Chris Richardson, *Microservices Patterns* — Transactional Outbox
- microservices.io/patterns/data/transactional-outbox.html
