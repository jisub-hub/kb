---
tags:
  - pattern
  - design-pattern
  - persistence
  - ddd
created: 2026-06-15
---

# Repository Pattern (리포지토리 패턴)

> [!summary] 한 줄 요약
> 도메인 객체의 **영속성(저장/조회) 로직을 추상화**하여, 도메인이 마치 **메모리 컬렉션**을 다루듯 객체를 다루게 하는 패턴.

---

## 1. 개념

- 도메인 계층과 데이터 접근 계층 사이의 **중재자**.
- 도메인 코드는 SQL/ORM/쿼리 세부사항을 모른다 → **인프라 의존성 분리**.
- [[DDD]]에서는 **애그리거트 루트 단위**로만 리포지토리를 둔다.

```java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
    void delete(Order order);
}
// 도메인은 이 인터페이스에만 의존, 구현은 Infrastructure 계층
```

---

## 2. DAO 와의 차이

| | Repository | DAO |
|---|------------|-----|
| 관점 | **도메인 중심** (애그리거트 컬렉션) | **데이터/테이블 중심** |
| 반환 | 도메인 객체(애그리거트) | 테이블 행/DTO |
| 추상화 수준 | 높음 (영속성 은닉) | 낮음 (CRUD 매핑) |

> 실무에선 혼용되지만, 개념적으로 Repository는 도메인 모델의 일부에 가깝다.

---

## 3. 장점 / 단점

### ✅ 장점
- 영속성 기술 교체 용이 (DB ↔ 메모리 ↔ 다른 ORM).
- **테스트 용이** — 인메모리 구현/목으로 대체.
- 도메인 로직과 쿼리 로직의 **관심사 분리**.

### ❌ 단점
- 단순 CRUD에선 **불필요한 추상화 레이어**.
- 복잡한 조회를 모두 리포지토리에 넣으면 비대해짐 → 조회는 [[CQRS]]의 Query 측으로 분리하기도.

---

## 4. 주의점 / 팁

- 리포지토리에 **조회 메서드를 무한정 추가**하지 말 것 → CQRS의 read 모델/쿼리 서비스로 분리 고려.
- 인터페이스는 **도메인 계층**, 구현체는 **인프라 계층** ([[Hexagonal-Architecture]] 의 Port/Adapter).

---

## 6. Spring 구현 예시

### 6.1. 가장 단순한 형태 — Spring Data JPA
```java
@Entity
public class Member {
    @Id @GeneratedValue private Long id;
    private String email;
    private String name;
}

// 인터페이스만 선언하면 Spring이 구현체를 런타임에 생성
public interface MemberRepository extends JpaRepository<Member, Long> {
    Optional<Member> findByEmail(String email);          // 쿼리 메서드 자동 생성
    List<Member> findByNameContaining(String keyword);
}
```

### 6.2. DDD 스타일 — 도메인 인터페이스 + 인프라 구현 분리
[[Hexagonal-Architecture]]의 Port/Adapter처럼, **인터페이스는 도메인 계층, 구현은 인프라 계층**에 둔다.
```java
// ── domain 계층 (기술 모름) ──────────────────────────
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    Order save(Order order);
}

// ── infrastructure 계층 ──────────────────────────────
// 방법 A) Spring Data 인터페이스를 도메인 인터페이스가 상속
public interface JpaOrderRepository
        extends OrderRepository, JpaRepository<Order, OrderId> {}

// 방법 B) 어댑터로 감싸기 (도메인을 Spring Data에 묶지 않음, 더 순수)
@Repository
@RequiredArgsConstructor
public class OrderRepositoryAdapter implements OrderRepository {
    private final OrderJpaDao jpaDao;        // 내부 Spring Data DAO

    @Override public Optional<Order> findById(OrderId id) {
        return jpaDao.findById(id.value()).map(OrderEntity::toDomain);
    }
    @Override public Order save(Order order) {
        return jpaDao.save(OrderEntity.fromDomain(order)).toDomain();
    }
}
```

### 6.3. 테스트 — 인메모리 구현으로 대체
```java
// 도메인 인터페이스 덕분에 DB 없이 테스트 가능
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> store = new ConcurrentHashMap<>();
    @Override public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id));
    }
    @Override public Order save(Order order) {
        store.put(order.getId(), order); return order;
    }
}

@Test
void 주문_저장_조회() {
    OrderRepository repo = new InMemoryOrderRepository();   // 빠른 단위 테스트
    Order order = Order.place(new CustomerId("c1"), List.of());
    repo.save(order);
    assertThat(repo.findById(order.getId())).isPresent();
}
```

> [!tip] 조회 폭증 시
> `findByXxx` 가 끝없이 늘면 [[CQRS]]의 Query 측(전용 조회 서비스/프로젝션)으로 분리하라. 쓰기 리포지토리는 애그리거트 단위로 단순하게 유지.

---

## 5. 관련 패턴

- [[DDD]] — 애그리거트 루트 단위 리포지토리
- [[CQRS]] — 복잡한 조회는 Query 측 분리
- [[Hexagonal-Architecture]] — Port(인터페이스)/Adapter(구현)

## 참고
- Martin Fowler, *PoEAA* — Repository
