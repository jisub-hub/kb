---
tags:
  - database
  - jpa
  - hibernate
  - orm
  - sync
created: 2026-06-15
---

# JPA / Hibernate

> [!summary] 한 줄 요약
> **JPA(Java Persistence API)**: 자바 ORM **표준 스펙**. **Hibernate**: 그 대표 구현체. 객체와 테이블을 매핑해 SQL을 자동 생성하고, **영속성 컨텍스트**로 객체 상태를 관리한다.

---

## 1. 특징
- **동기/블로킹**, [[JDBC]] 기반.
- **ORM**: 객체 ↔ 테이블 매핑, SQL 자동 생성(JPQL/메서드 쿼리).
- **영속성 컨텍스트(1차 캐시)** + **더티 체킹** + **지연 로딩** → 생산성↑.
- [[DDD]] 애그리거트의 Write 모델 영속화에 잘 맞음.

---

## 2. 엔티티 & 연관관계
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)          // 지연 로딩 (기본 권장)
    @JoinColumn(name = "member_id")
    private Member member;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status;
}
```

## 3. Spring Data JPA — Repository
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    // 메서드 이름으로 쿼리 자동 생성
    List<Order> findByMemberIdAndStatus(Long memberId, OrderStatus status);

    // JPQL (엔티티 대상 쿼리)
    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")  // N+1 방지 fetch join
    Optional<Order> findWithLines(@Param("id") Long id);
}
```

## 4. 영속성 컨텍스트 & 더티 체킹
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;

    @Transactional
    public void cancel(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel();           // ⭐ save() 호출 없이도 트랜잭션 커밋 시 UPDATE 자동 실행 (더티 체킹)
    }
}
```

## 5. ⚠️ 주의해야 할 함정
| 함정 | 설명 / 대응 |
|------|------------|
| **N+1 문제** | 연관 엔티티를 루프에서 개별 조회 → `fetch join`, `@EntityGraph`, batch size로 해결 (자세히는 **6장**) |
| **LazyInitializationException** | 트랜잭션 밖에서 지연 로딩 접근 → DTO 변환/페치 조인으로 미리 로딩 |
| **OSIV (Open Session In View)** | 기본 true → 커넥션 오래 점유. 운영에선 `false` 권장 (자세히는 **7장**) |
| **양방향 연관관계 무한루프** | 엔티티 직접 직렬화 금지 → **DTO 반환** |
| **엔티티를 API 응답으로 노출** | 결합/보안 문제 → DTO/Projection 사용 |

## 6. N+1 문제 — 원리부터 해결까지

JPA 성능 함정의 대표격. **연관 엔티티를 컬렉션/객체 단위로 루프에서 접근**할 때, 최초 1번의 조회 쿼리 외에 연관 데이터를 채우려고 **N번의 추가 쿼리**가 나가는 현상.

### 6.1 발생 원리

```java
// 1번 쿼리: orders 100건 조회
List<Order> orders = orderRepository.findAll();          // SELECT * FROM orders

for (Order order : orders) {
    // 지연 로딩(LAZY) 프록시 초기화 → order마다 SELECT 발생
    order.getMember().getName();                          // SELECT * FROM member WHERE id = ?  (×100)
    order.getLines().size();                              // SELECT * FROM order_line WHERE order_id = ?  (×100)
}
// 총 1 + 100(member) + 100(lines) = 201 쿼리 → "1 + N"
```

- **LAZY**라서 루프 안에서 프록시/컬렉션을 처음 건드릴 때마다 쿼리가 나간다.
- **EAGER로 바꿔도 해결되지 않는다.** EAGER는 즉시 로딩일 뿐, `findAll()` 같은 JPQL은 여전히 각 연관관계를 개별 쿼리로 채운다(오히려 의도치 않은 시점에 N+1 유발). 그래서 **기본은 LAZY**, 필요 시점에만 함께 로딩하는 게 정석.

### 6.2 탐지 방법

| 방법 | 설명 |
|------|------|
| `spring.jpa.properties.hibernate.show_sql=true` + `format_sql=true` | 콘솔에 실제 SQL 출력. 같은 모양 쿼리가 반복되면 N+1 의심 |
| **p6spy** (`p6spy-spring-boot-starter`) | 바인딩 파라미터까지 포함한 실제 쿼리·실행시간 로깅 |
| **datasource-proxy** | 쿼리 카운트/슬로우 쿼리 후킹, 커스텀 리스너 |
| **쿼리 카운트 assert** | 테스트에서 발생 쿼리 수를 검증 → 회귀 방지 |

```java
// 테스트에서 쿼리 수를 고정 (Hibernate Statistics 활용)
@Test
void noNPlusOne() {
    Statistics stats = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
    stats.setStatisticsEnabled(true);
    stats.clear();

    List<Order> orders = orderRepository.findAllWithLines();   // fetch join 적용 버전
    orders.forEach(o -> o.getLines().size());

    assertThat(stats.getPrepareStatementCount()).isEqualTo(1); // 쿼리 1번이어야 통과
}
```

### 6.3 해결책 비교

| 방법 | 메커니즘 | 페이징 | 컬렉션 다중 | 비고 |
|------|----------|:------:|:-----------:|------|
| **fetch join (JPQL)** | `JOIN FETCH`로 한 방에 조인 조회 | ❌(컬렉션) | ❌ | 가장 강력하나 한계 있음(6.4) |
| **`@EntityGraph`** | 메서드/쿼리에 로딩할 연관관계 선언, 내부적으로 fetch join | ❌(컬렉션) | ❌ | JPQL 안 쓰고 선언적으로 |
| **`@BatchSize` / `default_batch_fetch_size`** | LAZY 로딩을 모아서 `IN (?, ?, …)` 한 방에 | ✅ | ✅ | N+1 → **N/batch + 1**로 완화 |
| **FetchType 조정** | 꼭 필요한 연관만 로딩 전략 점검 | — | — | 보통 LAZY 유지가 정답 |
| **DTO projection** | 필요한 컬럼만 `new DTO(...)` / 인터페이스 프로젝션 | ✅ | ✅ | 엔티티 영속화 불필요한 조회에 최적 |

```java
// (1) fetch join — 컬렉션 1개를 즉시 로딩
@Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
Optional<Order> findWithLines(@Param("id") Long id);

// (2) @EntityGraph — JPQL 없이 선언적으로
@EntityGraph(attributePaths = {"member", "lines"})
List<Order> findByStatus(OrderStatus status);

// (3) DTO projection — 조회 전용, 영속성 컨텍스트 부담 없음
@Query("SELECT new com.example.OrderSummary(o.id, m.name, COUNT(l)) " +
       "FROM Order o JOIN o.member m JOIN o.lines l GROUP BY o.id, m.name")
List<OrderSummary> findSummaries();
```

### 6.4 fetch join의 한계

- **컬렉션 둘 이상 fetch join 금지** → `MultipleBagFetchException`. `List`(=Bag) 컬렉션을 둘 이상 동시에 fetch join하면 카테시안 곱 + 중복 제어 불가로 예외. (회피책: `Set` 사용, 또는 하나는 fetch join + 나머지는 `@BatchSize`)
- **컬렉션 fetch join + 페이징 불가** → 1:N 조인은 결과 row가 뻥튀기되어 DB 레벨 페이징(`limit`)을 적용할 수 없다. Hibernate는 경고(`HHH000104: firstResult/maxResults specified with collection fetch; applying in memory`)를 남기고 **전체를 메모리로 읽은 뒤 메모리에서 페이징** → 대용량이면 OOM 위험.

```
⚠️ 컬렉션 fetch join + Pageable 조합은 "메모리 페이징" 경고를 의심하라.
   → ToOne 관계만 fetch join하고, 컬렉션은 default_batch_fetch_size로 처리.
```

### 6.5 @BatchSize / default_batch_fetch_size

LAZY 로딩 시 연관 키를 **모아서 `IN` 절 한 방**으로 조회 → `1 + N`을 `1 + ceil(N/batch)`로 줄인다. 컬렉션 fetch join이 막히는 페이징 상황의 현실적 해답.

```yaml
# application.yml — 전역 적용 (실무 권장: 100~1000)
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

```java
// 특정 연관관계에만 적용
@OneToMany(mappedBy = "order")
@BatchSize(size = 100)
private List<OrderLine> lines = new ArrayList<>();
// 결과: SELECT * FROM order_line WHERE order_id IN (?, ?, ... 최대 100개)
```

### 6.6 실무 권장 정리

1. **기본은 모두 LAZY** — EAGER는 예측 불가능한 N+1의 원인.
2. **단일 조회 / ToOne 관계**: 필요하면 `fetch join` 또는 `@EntityGraph`로 한 번에.
3. **컬렉션 + 페이징**: 컬렉션은 fetch join하지 말고 `default_batch_fetch_size`(또는 `@BatchSize`)로.
4. **조회 전용 화면**: 엔티티 대신 **DTO projection**으로 필요한 컬럼만.
5. 테스트에 **쿼리 카운트 assert**를 걸어 N+1 회귀를 막는다.

## 7. 프로덕션 튜닝 트레이드오프

N+1(6장)은 "쿼리 수"의 문제였다. 여기서는 **고동시성·대량 처리 환경에서 드러나는 트레이드오프** — 커넥션 점유, 메모리/GC, 캐시 일관성 — 를 "X vs Y를 언제 쓰는가" 기준으로 정리한다.

### 7.1 OSIV(Open Session In View)

`spring.jpa.open-in-view`의 **기본값은 `true`**. 영속성 컨텍스트(와 그것이 잡은 **DB 커넥션**)를 트랜잭션 종료가 아니라 **HTTP 응답이 뷰까지 렌더링되는 시점**까지 열어 둔다. 컨트롤러·뷰에서 지연 로딩이 자유롭다는 게 유일한 장점이지만, 그 대가가 크다.

```
OSIV=true 요청 생명주기 (커넥션 점유 구간 = ▓)
  요청 ─▓▓▓ @Transactional 서비스 ▓▓▓─ 컨트롤러 ─ JSON 직렬화 ─ 응답
        └──────────────── 커넥션을 응답 끝까지 보유 ────────────────┘

OSIV=false
  요청 ─▓▓▓ @Transactional 서비스 ▓▓▓─┘ (여기서 커넥션 즉시 반납)
                                      컨트롤러·직렬화는 커넥션 없이
```

**고동시성에서 무엇이 터지나**:
- 트랜잭션이 끝나도 커넥션을 반납하지 않으므로, **외부 API 호출·느린 뷰 렌더 중에도 커넥션을 점유**한다. 응답이 200ms인데 그중 150ms가 외부 호출이면, 커넥션을 150ms 동안 놀리며 잡고 있는 셈.
- 커넥션 풀(HikariCP 기본 10)이 빠르게 고갈 → 후속 요청이 `Connection is not available` 타임아웃.

| | OSIV `true` (기본) | OSIV `false` (운영 권장) |
|---|---|---|
| 커넥션 점유 | 응답 끝까지 | 트랜잭션 종료 즉시 반납 |
| 컨트롤러/뷰 지연 로딩 | 가능 | `LazyInitializationException` |
| 고동시성 풀 고갈 | 위험 ↑ | 안전 |
| 개발 편의 | 높음 | 명시적 fetch 필요 |

**언제 무엇을** — 프로토타입/사내 어드민처럼 동시성이 낮으면 `true`로 두고 편하게. **트래픽 있는 운영 서비스는 `false`**로 두고, 트랜잭션 안에서 fetch join/`@EntityGraph`/DTO로 필요한 것을 **명시적으로 미리 로딩**한다(6장과 연결).

```yaml
# application.yml — 운영 권장
spring:
  jpa:
    open-in-view: false   # 켜져 있으면 부팅 시 warn 로그도 남는다
```

### 7.2 batch fetch vs fetch join — 메모리/GC 관점

둘 다 N+1을 잡지만 **메모리 프로파일이 정반대**다.

| | fetch join | batch fetch (`default_batch_fetch_size`) |
|---|---|---|
| 쿼리 수 | 1 | `1 + ceil(N / batch)` |
| 1:N 결과 형태 | **카테시안 곱**(부모 × 자식 row) | 부모 1쿼리 + 자식 `IN` 쿼리 |
| 페이징 | **컬렉션이면 불가**(메모리 페이징) | 가능 |
| 메모리 압박 | 큰 결과셋에서 row 폭증 → JDBC 결과·중복 엔티티로 힙 압박 | 부모만큼만, 자식은 배치 단위로 |

**수치 예시** — 주문 1,000건, 각 주문 평균 10개 라인:
- **fetch join**: `1,000 × 10 = 10,000 row`가 한 ResultSet으로. Hibernate가 중복 부모를 메모리에서 distinct 처리 → 순간 힙 점유 큼, 페이징도 불가(메모리 페이징 경고).
- **batch(size=100)**: 부모 1쿼리(1,000건) + 자식 `1,000/100 = 10`쿼리. 총 11쿼리, ResultSet은 작게 쪼개져 GC 압박이 완만.

판단 기준:
- **단건/소수 + ToOne 위주** → fetch join (왕복 1번이 최선).
- **대량 + 컬렉션 + 페이징 필요** → batch fetch. fetch join은 페이징이 막히고 카테시안 곱으로 힙이 터진다.

### 7.3 2차 캐시(L2)의 함정

1차 캐시는 트랜잭션 단위라 안전하지만, **2차 캐시는 SessionFactory(앱 전체) 단위**라 동시성·분산·신선도 문제를 동반한다.

- **Cache stampede(미스 폭주)**: 인기 키의 캐시가 만료되는 순간, 동시 요청 N개가 전부 미스 → DB로 N개 쿼리가 동시에 쏟아진다. (완화: 만료 시 단일 로더만 채우는 lock/early-recompute)
- **Stale read**: 읽기 전용으로 캐시한 엔티티를 외부 배치/다른 앱이 DB에서 직접 수정하면, 캐시는 옛 값을 계속 반환. JPA를 안 거친 변경은 캐시가 모른다.
- **분산 무효화 일관성**: 여러 인스턴스가 각자 로컬 L2 캐시를 들면, 한 노드의 수정이 다른 노드 캐시를 무효화하지 못한다. 분산 캐시(예: Redis/Hazelcast 백엔드)나 무효화 전파가 필요하고, 그 자체가 새 일관성 문제를 만든다.

**언제 쓰지 말아야 하나**:
- 쓰기가 잦은 엔티티 (무효화 비용 > 캐시 이득).
- JPA 외 경로(네이티브 배치, 타 서비스)로 같은 테이블을 수정하는 경우 → stale 보장.
- 다중 인스턴스인데 무효화 전파 인프라가 없을 때.
- **권장 대상**: 거의 안 바뀌는 코드성/참조 데이터(국가·코드표 등) 정도로 좁게. 그 외 변동 데이터는 애플리케이션 레벨 캐시(Cache-Aside, TTL 짧게)가 통제하기 쉽다.

### 7.4 쓰기 지연 · flush 타이밍

JPA는 변경을 모았다가 **flush 시점에 한꺼번에** SQL을 내보낸다(쓰기 지연). 대량 처리에서 두 가지가 문제된다.

**영속성 컨텍스트 크기 폭발** — 루프에서 수만 건을 persist하면 1차 캐시에 전부 쌓여 힙이 터지고 더티 체킹 대상도 선형으로 늘어 느려진다. **batch 단위로 `flush()` + `clear()`**:

```java
@Transactional
public void bulkInsert(List<Order> orders) {
    for (int i = 0; i < orders.size(); i++) {
        em.persist(orders.get(i));
        if (i % 500 == 0) {     // 500건마다 비우기
            em.flush();          // DB로 내보내고
            em.clear();          // 영속성 컨텍스트 비워 힙·더티체킹 부담 제거
        }
    }
}
```

**batch insert가 실제로 묶이게** — `IDENTITY` 전략은 INSERT 즉시 PK를 받아와야 해서 JDBC batch가 사실상 비활성된다. batch insert가 필요하면 `SEQUENCE`(또는 table) 전략을 쓰고 batch size를 설정한다. PostgreSQL은 `reWriteBatchedInserts=true`로 다건 INSERT를 단일 multi-row INSERT로 재작성해 처리량이 크게 오른다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 500
        order_inserts: true     # 같은 테이블 INSERT를 모아 batch 효율↑
        order_updates: true
  datasource:
    # PostgreSQL: INSERT ... VALUES (...),(...),(...) 로 재작성
    url: jdbc:postgresql://.../db?reWriteBatchedInserts=true
```

판단 기준: 진짜 대량(수십만 건+) 적재라면 JPA보다 **JDBC batch / `JdbcTemplate` / bulk COPY**가 정답인 경우가 많다. JPA batch는 "엔티티 생명주기·이벤트가 필요한 중간 규모"에 적합.

## 8. 복잡 동적 쿼리 → QueryDSL
```java
// 타입 세이프 동적 쿼리 ([[QueryDSL]])
List<Order> result = queryFactory
    .selectFrom(order)
    .where(
        memberIdEq(cond.memberId()),     // null이면 조건 무시 (BooleanExpression)
        statusEq(cond.status()))
    .orderBy(order.id.desc())
    .fetch();
```

## 9. 장점 / 단점
### ✅ 장점
- **생산성 매우 높음** (CRUD 자동, SQL 최소화).
- 객체지향적 도메인 모델링, DB 벤더 독립성(방언).
- 1차 캐시·더티체킹·지연로딩 등 풍부한 기능.

### ❌ 단점
- **학습 곡선 가파름**, 내부 동작 이해 없이 쓰면 성능 함정(N+1 등).
- 복잡한 통계/리포팅 쿼리엔 불리 → MyBatis/QueryDSL/Native 병행.

## 10. 관련
- [[JDBC]] · [[MyBatis]] · [[JdbcTemplate]] · [[QueryDSL]]
- [[DDD]] · [[Repository-Pattern]] · [[CQRS]]
- [[Transaction-Management]] — `@Transactional` 전파·격리·flush(readOnly)·롤백 규칙
- 비동기 대응: [[R2DBC]]
