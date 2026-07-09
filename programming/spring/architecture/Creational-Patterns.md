---
tags:
  - pattern
  - design-pattern
  - creational
  - gof
created: 2026-06-17
---

# 생성 패턴 (Creational Patterns) — GoF

> [!summary] 한 줄 요약
> **객체 생성 과정을 캡슐화**해, 무엇을·어떻게·언제 만들지를 사용 코드와 분리하는 생성(creational) 패턴군. GoF의 5가지(Singleton·Factory Method·Abstract Factory·Builder·Prototype) 중 Spring 실무에서 핵심은 **Singleton·Factory Method·Builder**다 — Spring 컨테이너 자체가 거대한 생성 패턴 구현체([[Spring-Core]])이며, `@Bean`·`BeanFactory`·`@Builder`로 매일 만난다.

---

## 1. Singleton — 인스턴스 단 하나 ⭐

**개념**: 클래스의 인스턴스를 **JVM 내 하나만** 보장하고, 전역 접근점을 제공한다. 공유 자원(설정·캐시·커넥션 풀)에 쓰지만, 직접 구현 시 **thread-safety**와 **전역 상태(global state)** 안티패턴이 함정이다.

직접 구현은 enum 또는 holder idiom이 정석이다 — `synchronized`나 DCL(double-checked locking)은 장황하고 실수하기 쉽다.

```java
// ① enum 싱글톤 — 가장 견고 (직렬화·리플렉션 공격에도 안전, Effective Java 권장)
public enum PaymentConfig {
    INSTANCE;
    private final String apiKey = load();
    public String apiKey() { return apiKey; }
}

// ② Lazy holder idiom — 클래스 로딩 시점에 thread-safe하게 초기화 (지연 로딩 + 동기화 비용 0)
public class ConnectionPool {
    private ConnectionPool() {}
    private static class Holder {            // 최초 getInstance() 호출 시에만 로딩
        private static final ConnectionPool INSTANCE = new ConnectionPool();
    }
    public static ConnectionPool getInstance() { return Holder.INSTANCE; }
}
```

> JVM의 클래스 초기화는 그 자체로 thread-safe하다. Holder는 이 보장을 이용해 동기화 없이 지연 초기화를 얻는다.

**Spring/실무에서의 활용**: Spring Bean의 **기본 스코프가 사실상 싱글톤**이다 — 단, GoF 싱글톤(JVM당 1개, static 접근)과는 다르다. Spring은 **컨테이너(ApplicationContext)당 1개**를 컨테이너가 관리하며, 사용 측은 static 접근이 아니라 **DI로 주입**받는다. 이 차이가 핵심이다:

| | GoF Singleton | Spring Singleton Bean |
|--|---------------|----------------------|
| 범위 | JVM당 1개 | **컨테이너당 1개** (컨테이너 여러 개면 여러 개) |
| 접근 | `getInstance()` static 전역 | **DI 주입** (생성자/필드) |
| 테스트 | 교체 어려움(전역 상태) | 목 주입 쉬움 |
| 생명주기 | 직접 관리 | 컨테이너가 관리(@PostConstruct 등) |

> [!tip] 싱글톤 빈 ≠ 무상태 강제
> 싱글톤 빈을 여러 스레드가 공유하므로 **가변 인스턴스 필드는 동시성 버그**가 된다. 빈은 무상태(stateless)로 두고, 요청별 상태는 메서드 파라미터·`ThreadLocal`·request 스코프로 넘긴다.

**언제 쓰나**: 전역적으로 공유되는 무상태 컴포넌트(서비스·설정·풀). 직접 GoF 싱글톤을 손으로 구현할 일은 드물다 — **Spring을 쓴다면 빈으로 등록**하면 끝이다. 직접 구현은 전역 상태를 만들어 테스트·결합을 해치므로 안티패턴으로 취급한다.

---

## 2. Factory Method — 생성을 서브클래스에 위임 ⭐

**개념**: 객체 생성을 위한 **인터페이스(팩토리 메서드)** 만 정의하고, **어떤 구체 타입을 만들지는 서브클래스가 결정**한다. `new`를 사용 코드에서 걷어내, 생성 책임을 한 곳으로 모은다.

```
[Creator] ──factoryMethod()──► Product (추상)
    ▲                              ▲
    │ 오버라이드                    │ 구현
[ConcreteCreator] ──생성──► [ConcreteProduct]
```

```java
// Product 추상 + Creator가 생성을 추상 메서드로 위임
abstract class Notifier {
    void send(String msg) {                  // 공통 흐름 (템플릿)
        Channel ch = createChannel();        // ← 무엇을 만들지는 서브클래스가 결정
        ch.deliver(msg);
    }
    protected abstract Channel createChannel();   // 팩토리 메서드
}

class SmsNotifier extends Notifier {
    protected Channel createChannel() { return new SmsChannel(); }
}
class EmailNotifier extends Notifier {
    protected Channel createChannel() { return new EmailChannel(); }
}
```

**Spring/실무에서의 활용**: Spring 컨테이너 자체가 거대한 팩토리다.

```java
// ① @Bean 메서드 = 팩토리 메서드. "이 타입을 어떻게 만드는가"를 메서드로 캡슐화
@Configuration
class AppConfig {
    @Bean
    PaymentGateway paymentGateway(PgProperties props) {   // 생성 로직 + 의존성 조립
        return new TossPaymentGateway(props.apiKey());
    }
}
```

- **`BeanFactory`** — Spring DI 컨테이너의 최상위 인터페이스 이름 자체가 팩토리다. `getBean(Class)`는 팩토리 메서드 호출과 같다.
- **`FactoryBean<T>`** — 빈 하나의 생성 로직이 복잡할 때(프록시·커넥션 등) 그 객체를 만드는 팩토리를 빈으로 등록한다. `getObject()`가 팩토리 메서드. `SqlSessionFactoryBean`(MyBatis), `LocalContainerEntityManagerFactoryBean`(JPA)이 대표 예.
- **`@Bean` 메서드** — 가장 흔한 실무 형태. 정적 팩토리(`List.of`, `Optional.of`)도 같은 정신.

> 정적 팩토리 메서드(`of`, `from`, `valueOf`)는 엄밀히 GoF Factory Method는 아니지만 같은 가치를 준다 — 이름으로 의도 표현, 인스턴스 캐싱(`Integer.valueOf`), 반환 타입 유연성.

**언제 쓰나**: 생성할 구체 타입이 **런타임/하위 클래스에 따라 달라질 때**, `new`를 흩뿌리지 않고 생성 책임을 한 지점에 모으고 싶을 때. Spring에선 사실상 `@Bean`으로 일상적으로 사용 중이다.

---

## 3. Abstract Factory — 관련 객체군 생성

**개념**: 서로 **관련된 객체들의 묶음(제품군)** 을, 구체 클래스를 노출하지 않고 생성한다. "팩토리를 만드는 팩토리" — 한 팩토리가 일관된 한 벌의 제품을 함께 만든다.

```java
// 제품군: DB별로 Connection + Dialect가 한 벌로 묶임
interface DbFactory {
    Connection connection();
    Dialect dialect();
}
class PostgresFactory implements DbFactory {        // Pg 한 벌
    public Connection connection() { return new PgConnection(); }
    public Dialect dialect()       { return new PgDialect(); }
}
// 사용 측은 구체 타입을 모른 채 일관된 한 벌을 받는다 (Pg + Pg, MySql + MySql, 섞이지 않음)
class QueryRunner { QueryRunner(DbFactory f) { conn = f.connection(); dialect = f.dialect(); } }
```

**Spring/실무에서의 활용**: 순수 GoF Abstract Factory를 직접 구현하는 일은 흔치 않다 — 대부분 **DI + 프로파일/조건부 빈**으로 대체한다. `@Profile("postgres")`로 제품군 전체를 묶어 등록하거나, `@ConditionalOnProperty`로 한 벌의 빈 그룹을 통째로 교체하는 것이 Spring식 Abstract Factory다.

**언제 쓰나**: 여러 객체가 **반드시 같은 계열로 함께 묶여야** 할 때(섞이면 안 됨). 단일 객체만 다양하면 Factory Method로 충분하다.

---

## 4. Builder — 복잡한 객체를 단계적으로 ⭐

**개념**: 생성자 인자가 많거나 선택적일 때, **단계별로 설정한 뒤 마지막에 조립**한다. 핵심 동기는 **텔레스코핑 생성자(telescoping constructor) 문제** 해결과 **불변(immutable) 객체** 생성이다.

```java
// 문제: 텔레스코핑 생성자 — 인자 순서/의미를 알 수 없고 조합마다 오버로드 폭발
new Order(id, customerId, null, items, null, true, null);  // 이 null들이 다 뭔가?

// Builder: 이름 있는 단계 + 불변 결과
Order order = Order.builder()
    .customerId("C-1")
    .addItem(item)
    .coupon(coupon)        // 선택값만 명시
    .build();              // 이 시점에 검증 + 불변 객체 생성
```

```java
// 직접 구현 (Lombok 없이) — 핵심 구조
public final class Order {
    private final String customerId;
    private final List<OrderItem> items;
    private Order(Builder b) {                    // private 생성자
        this.customerId = Objects.requireNonNull(b.customerId);
        this.items = List.copyOf(b.items);        // 방어적 복사 → 불변
    }
    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String customerId;
        private final List<OrderItem> items = new ArrayList<>();
        public Builder customerId(String v) { this.customerId = v; return this; }  // 체이닝
        public Builder addItem(OrderItem i)  { this.items.add(i); return this; }
        public Order build() {
            if (items.isEmpty()) throw new IllegalStateException("빈 주문");  // 불변식 검증
            return new Order(this);
        }
    }
}
```

**Spring/실무에서의 활용**:

- **Lombok `@Builder`** — 위 보일러플레이트를 어노테이션 하나로 생성한다. 불변 DTO/도메인 객체([[DTO]])에 `@Builder` + `@Value`(또는 record + `@Builder`) 조합이 실무 표준.
- **`@Builder.Default`** — 빌더 사용 시 필드 기본값을 보존(설정 안 한 필드가 null이 되는 함정 방지).
- 프레임워크 곳곳에 빌더가 있다: `UriComponentsBuilder`, `RestClient.builder()`, `WebClient.builder()`, `MockMvcRequestBuilders`, `Stream.builder()`.

> [!tip] record와 Builder
> Java `record`는 불변·간결하지만 **선택적 필드가 많으면 생성자 호출이 장황**해진다. 필드 3~4개 이하면 record 정적 팩토리로 충분, 그 이상·선택값 多이면 `@Builder`를 얹는다.

**언제 쓰나**: 생성자 인자가 **4개 이상**이거나 **선택적 인자가 많을 때**, **불변 객체**를 만들면서 가독성을 지키고 싶을 때, 생성 시점에 **불변식 검증**을 모으고 싶을 때.

---

## 5. Prototype — 복제로 생성

**개념**: `new`로 처음부터 만드는 대신, **기존 인스턴스를 복제(clone)** 해 새 객체를 얻는다. 생성 비용이 크거나(무거운 초기화), 런타임에 결정된 상태를 가진 객체를 찍어낼 때 쓴다.

```java
// 복사 생성자 방식 (Cloneable/Object.clone()보다 권장)
public final class SearchQuery {
    private final List<String> filters;
    public SearchQuery(SearchQuery src) { this.filters = new ArrayList<>(src.filters); } // 깊은 복사
    public SearchQuery withFilter(String f) { var c = new SearchQuery(this); c.filters.add(f); return c; }
}
```

> Java의 `Cloneable`/`Object.clone()`은 깨진 설계(얕은 복사·예외·계약 모호)로 악명 높다. 실무에선 **복사 생성자나 정적 `copyOf`** 를 쓴다.

**Spring/실무에서의 활용**: 여기서 **이름 충돌 주의** — Spring의 `@Scope("prototype")`은 GoF Prototype 패턴과 **이름만 같고 메커니즘이 다르다**. Spring prototype 스코프는 **요청할 때마다 `new`로 새 인스턴스를 생성**하는 것이지, 기존 객체를 **복제**하는 것이 아니다.

| | GoF Prototype | Spring prototype 스코프 |
|--|---------------|------------------------|
| 메커니즘 | 기존 객체 **복제(clone)** | 요청마다 **새로 생성(new)** |
| 목적 | 생성 비용 절감·상태 복사 | 빈마다 독립 인스턴스 보장 |
| 생명주기 | — | 생성 후 컨테이너가 **소멸 관리 안 함** |

**언제 쓰나(GoF)**: 객체 생성 비용이 크고 **유사한 객체를 다량 찍어낼 때**, 또는 동적으로 구성된 객체를 원형으로 두고 변형본을 만들 때. 현대 Java에선 빈도가 낮다.

---

## 6. 정리 — 한눈 비교

| 패턴 | 한 줄 정의 | Spring 실무 접점 |
|------|-----------|-----------------|
| **Singleton** | 인스턴스 1개 보장 | 빈 기본 스코프(컨테이너 관리), DI |
| **Factory Method** | 생성을 서브클래스/메서드에 위임 | `@Bean`, `BeanFactory`, `FactoryBean` |
| **Abstract Factory** | 관련 객체군을 한 벌로 생성 | `@Profile`/조건부 빈 그룹 |
| **Builder** | 복잡·불변 객체 단계적 조립 | Lombok `@Builder`, `*Builder` API |
| **Prototype** | 복제로 생성 | (이름만 같은 prototype 스코프와 구분) |

> 핵심 통찰: **Spring 컨테이너 = 생성 패턴의 종합 구현체**. 싱글톤 관리·팩토리(@Bean)를 프레임워크가 떠안으므로, 애플리케이션 코드가 직접 작성하는 생성 패턴은 사실상 **Builder** 하나로 수렴한다. 나머지는 "프레임워크가 이미 해주는 것"으로 이해하는 게 실무 관점이다.

---

## 관련
- [[Spring-Core]] — DI 컨테이너가 싱글톤·팩토리를 관리
- [[Adapter-Pattern]] — 구조(structural) 패턴
- [[Structural-Patterns]] — GoF 구조 패턴 묶음
- [[Behavioral-Patterns]] — GoF 행위 패턴 묶음
- [[DTO]] — Builder로 만드는 대표 불변 객체

## 참고
- GoF, *Design Patterns* — Creational Patterns
- Joshua Bloch, *Effective Java* — Builder(아이템 2), enum 싱글톤(아이템 3), 정적 팩토리(아이템 1)
- refactoring.guru — Creational Patterns
