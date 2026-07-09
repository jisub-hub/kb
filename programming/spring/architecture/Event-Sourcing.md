---
tags:
  - pattern
  - architecture
  - event-sourcing
  - ddd
created: 2026-06-15
---

# Event Sourcing (이벤트 소싱)

> [!summary] 한 줄 요약
> 현재 **상태**를 직접 저장하는 대신, 상태를 변화시킨 **이벤트의 연속(append-only log)** 을 저장하고, 현재 상태는 이벤트들을 **재생(replay)** 해서 얻는다.

---

## 1. 개념

전통적 방식과 비교:

```
[전통적 CRUD]   계좌 잔액 = 50,000   (현재 상태만 저장, 과거는 사라짐)

[Event Sourcing]
   AccountOpened(0)
   Deposited(+30,000)
   Deposited(+40,000)
   Withdrawn(-20,000)
   ─────────────────────
   재생 결과 = 50,000   (모든 변화 이력이 영구 보존)
```

- 이벤트는 **불변(immutable)**, 추가만 가능(append-only).
- 이벤트는 **과거형**으로 명명: `OrderPlaced`, `MoneyDeposited`.

---

## 2. 핵심 구성요소

| 요소 | 설명 |
|------|------|
| **Event** | 일어난 사실. 불변. |
| **Event Store** | 이벤트를 순서대로 저장하는 append-only 저장소 |
| **Aggregate** | 이벤트를 적용해 현재 상태를 만들고, 명령 처리 시 새 이벤트 생성 ([[DDD]]) |
| **Projection** | 이벤트로부터 만든 **읽기 전용 뷰** (조회 최적화) |
| **Snapshot** | 특정 시점 상태를 저장해 재생 비용을 줄이는 최적화 |

```
Command ─► Aggregate.handle()
              │  (불변식 검증)
              ▼
          새 Event 생성 ─► Event Store(append)
                              │
                              ▼ (이벤트 구독)
                        Projection 갱신 ─► Read Model ([[CQRS]])
```

---

## 3. 장점 / 단점

### ✅ 장점
- **완전한 감사 이력(audit log)**: 모든 변경을 그대로 보존.
- **시점 복원(time travel)**: 과거 임의 시점 상태 재구성 가능.
- **디버깅/분석**: 무슨 일이 왜 일어났는지 추적.
- **이벤트 기반 통합**과 자연스럽게 연결 ([[CQRS]], [[Saga-Pattern]]).

### ❌ 단점
- **복잡도 大**: 학습·구현·운영 난이도 높음.
- **이벤트 스키마 진화** 어려움 (versioning, upcasting 필요).
- **조회는 Projection 필수** (이벤트 직접 조회는 비효율) → 사실상 CQRS 동반.
- **결과적 일관성** 수용 필요.

---

## 4. 언제 쓸 것인가

> [!success] 적합
> - 감사·이력·규제 추적이 중요한 도메인 (금융, 회계, 의료)
> - "왜 이 상태가 됐는가"가 비즈니스적으로 중요한 경우
> - 복잡한 도메인 + CQRS 적용 시

> [!failure] 부적합
> - 단순 CRUD, 이력이 필요 없는 경우
> - 팀이 패턴에 익숙하지 않고 빠른 출시가 우선일 때

---

## 5. 주의점

- **CQRS와 거의 항상 함께** 간다 (조회는 Projection으로).
- 이벤트는 **절대 수정/삭제하지 않는다** — 정정도 새 이벤트(보상 이벤트)로.
- **이벤트 버저닝 전략**을 처음부터 고민할 것.

---

## 7. Spring 구현 예시

> 프레임워크 없이 JPA로 직접 구현하는 최소 예시. (실무에선 **Axon Framework**, EventStoreDB 등을 많이 사용.)

### 7.1. 이벤트 정의
```java
public sealed interface AccountEvent
        permits AccountOpened, MoneyDeposited, MoneyWithdrawn {}

public record AccountOpened(UUID accountId, Instant at) implements AccountEvent {}
public record MoneyDeposited(UUID accountId, long amount, Instant at) implements AccountEvent {}
public record MoneyWithdrawn(UUID accountId, long amount, Instant at) implements AccountEvent {}
```

### 7.2. Event Store (append-only 테이블)
```java
@Entity
@Table(name = "event_store")
public class StoredEvent {
    @Id @GeneratedValue private Long seq;     // 글로벌 순번
    private UUID aggregateId;
    private long version;                      // 애그리거트 내 버전(낙관적 동시성)
    private String eventType;
    @Column(columnDefinition = "jsonb")
    private String payload;                    // 직렬화된 이벤트
    private Instant occurredAt;
}

public interface EventStoreRepository extends JpaRepository<StoredEvent, Long> {
    List<StoredEvent> findByAggregateIdOrderByVersionAsc(UUID aggregateId);
}
```

### 7.3. Aggregate — 이벤트 적용(apply) & 명령 처리
```java
public class Account {
    private UUID id;
    private long balance;
    private long version;
    private final List<AccountEvent> pendingEvents = new ArrayList<>();

    // (1) 명령 처리: 불변식 검증 후 새 이벤트 생성
    public void withdraw(long amount) {
        if (balance < amount) throw new IllegalStateException("잔액 부족");
        applyNew(new MoneyWithdrawn(id, amount, Instant.now()));
    }

    private void applyNew(AccountEvent e) { apply(e); pendingEvents.add(e); }

    // (2) 이벤트 적용: 상태 변경 로직 (재생 시에도 동일하게 사용)
    private void apply(AccountEvent e) {
        switch (e) {
            case AccountOpened o   -> { this.id = o.accountId(); this.balance = 0; }
            case MoneyDeposited d  -> this.balance += d.amount();
            case MoneyWithdrawn w  -> this.balance -= w.amount();
        }
        this.version++;
    }

    // (3) 재생(replay): 저장된 이벤트로 현재 상태 복원
    public static Account replay(List<AccountEvent> history) {
        Account a = new Account();
        history.forEach(a::apply);
        return a;
    }

    List<AccountEvent> pendingEvents() { return pendingEvents; }
}
```

### 7.4. Repository — 저장(append) / 로드(replay)
```java
@Repository
@RequiredArgsConstructor
public class AccountEventStore {
    private final EventStoreRepository store;
    private final ObjectMapper mapper;
    private final ApplicationEventPublisher publisher;

    @Transactional
    public void save(Account account) {
        for (AccountEvent e : account.pendingEvents()) {
            store.save(toStored(e));            // append-only
            publisher.publishEvent(e);          // Projection 갱신 트리거 → CQRS read model
        }
    }

    @Transactional(readOnly = true)
    public Account load(UUID accountId) {
        var events = store.findByAggregateIdOrderByVersionAsc(accountId)
                          .stream().map(this::toEvent).toList();
        return Account.replay(events);          // 이벤트 재생으로 상태 복원
    }
}
```

### 7.5. Projection — 읽기 모델 갱신 ([[CQRS]])
```java
@Component
@RequiredArgsConstructor
public class AccountBalanceProjection {
    private final AccountBalanceViewRepository viewRepo;  // 비정규화 조회 테이블

    @EventListener
    public void on(MoneyDeposited e) {
        viewRepo.addBalance(e.accountId(), e.amount());
    }
    @EventListener
    public void on(MoneyWithdrawn e) {
        viewRepo.addBalance(e.accountId(), -e.amount());
    }
}
```

> [!tip] 스냅샷
> 이벤트가 많아지면 `replay` 비용이 커진다. N개마다 현재 상태 스냅샷을 저장하고, 로드 시 *스냅샷 + 이후 이벤트만* 재생한다.

---

## 6. 관련 패턴

- [[CQRS]] — 조회용 Projection과 한 쌍
- [[DDD]] — 애그리거트가 이벤트 생성/적용
- [[Saga-Pattern]] — 이벤트 기반 분산 트랜잭션
- [[Outbox-Pattern]] — 이벤트 발행 신뢰성 보장

## 참고
- Greg Young — Event Sourcing 강연
- Martin Fowler, *Event Sourcing* — https://martinfowler.com/eaaDev/EventSourcing.html
