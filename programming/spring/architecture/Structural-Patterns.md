---
tags:
  - pattern
  - design-pattern
  - structural
  - gof
created: 2026-06-17
---

# 구조 패턴 (Structural Patterns) — GoF

> [!summary] 한 줄 요약
> **객체·클래스를 조합해 더 큰 구조를 만드는 방법**을 다루는 GoF 7개 패턴. 핵심은 "인터페이스를 어떻게 다루느냐"다 — **변환**(Adapter), **유지하며 감싸기**(Proxy·Decorator), **단순화**(Facade), **재귀 합성**(Composite), **추상-구현 분리**(Bridge), **공유**(Flyweight). Spring은 이 중 Proxy(AOP)·Decorator(BeanPostProcessor)·Facade(Service 계층)를 프레임워크 차원에서 적극 활용한다.

---

## 1. Adapter — 인터페이스 변환

호환되지 않는 두 인터페이스를 중간에서 변환해 잇는 패턴. Target(원하는 모양)과 Adaptee(기존 구현) 사이에 변환 계층을 끼운다. Spring·Hexagonal의 "Ports & Adapters"가 이를 아키텍처로 확장한 것.

> 상세는 [[Adapter-Pattern]] 참고. (Object Adapter, Hexagonal, 외부 SDK 격리 등)

---

## 2. Proxy — 같은 인터페이스로 접근 제어/지연/캐싱 ⭐

**대상 객체와 동일한 인터페이스**를 구현하면서, 실제 호출 전후에 부가 행위(접근 제어·지연 로딩·캐싱·로깅·트랜잭션)를 끼우는 패턴. 클라이언트는 진짜 객체인지 프록시인지 모른다 — 인터페이스가 같기 때문.

```java
public interface UserService {
    User find(Long id);
}

// 실제 구현
public class UserServiceImpl implements UserService { ... }

// 프록시 — 같은 인터페이스, 호출 전후 부가 행위
public class CachingUserProxy implements UserService {
    private final UserService target;
    private final Map<Long, User> cache = new ConcurrentHashMap<>();

    public User find(Long id) {
        return cache.computeIfAbsent(id, target::find); // 캐싱
    }
}
```

### Spring AOP @Transactional 동적 프록시 ⭐

Spring AOP는 **런타임 동적 프록시**로 구현된다. `@Transactional`·`@Async`·`@Cacheable` 같은 애너테이션은 빈 자체가 아니라, 빈을 감싼 **프록시**가 처리한다.

```
[Client] ──► [Proxy] ──트랜잭션 시작/커밋/롤백──► [실제 Bean]
                ▲
          @Transactional 처리는 여기서
```

프록시 생성 방식은 두 가지:

| 방식 | 조건 | 메커니즘 | 한계 |
|------|------|----------|------|
| **JDK Dynamic Proxy** | 대상이 **인터페이스** 구현 | `java.lang.reflect.Proxy`, 인터페이스 기반 | 인터페이스 없으면 불가 |
| **CGLIB** | 인터페이스 없음 / 강제 시 | **상속**으로 서브클래스 생성, 메서드 오버라이드 | `final` 클래스/메서드 프록시 불가, 기본 생성자 이슈 |

> Spring Boot는 기본적으로 **CGLIB**를 사용(`proxyTargetClass=true`가 기본). 인터페이스 유무와 무관하게 클래스 기반으로 일관 동작시키기 위함.

### @Transactional이 self-invocation에서 안 먹는 이유 ⭐

트랜잭션 처리는 **프록시를 거칠 때만** 작동한다. 같은 클래스 내부에서 메서드를 직접 호출(`this.xxx()`)하면 **프록시를 우회**하므로 애너테이션이 무시된다.

```java
@Service
public class OrderService {

    public void outer() {
        inner();   // ❌ this.inner() — 프록시 우회, @Transactional 안 먹음
    }

    @Transactional
    public void inner() {  // 외부에서 직접 호출될 때만 트랜잭션 적용
        ...
    }
}
```

`outer()`가 호출하는 `inner()`는 프록시가 아니라 **실제 객체 자신(this)** 을 가리킨다. 프록시는 외부 진입점에서만 끼어들기 때문에 내부 호출은 가로채지 못한다.

해결책:

| 방법 | 설명 |
|------|------|
| **메서드 분리** | `inner()`를 별도 빈으로 빼서 주입받아 호출 (프록시 경유) |
| **자기 자신 주입** | `@Lazy` 또는 `ApplicationContext`로 프록시 인스턴스 획득 후 호출 |
| **`AopContext.currentProxy()`** | `exposeProxy=true` 설정 후 프록시 직접 참조 (권장도 낮음) |
| **AspectJ 위빙** | 컴파일/로드타임 위빙은 프록시가 아니라 바이트코드 변경 → self-invocation도 적용 |

> 면접 단골: "@Transactional이 안 걸리는 경우?" → ① private 메서드 ② self-invocation ③ 체크 예외 기본 롤백 안 됨(`rollbackFor` 필요).

언제 쓰나:
```
✅ 원본을 건드리지 않고 접근 제어·캐싱·지연 로딩·로깅·트랜잭션을 횡단 적용
✅ 무거운 객체 생성을 실제 사용 시점까지 미룰 때 (JPA 지연 로딩 프록시)
⚠️ 인터페이스 vs 클래스 프록시 방식 차이, self-invocation 함정 인지 필수
```

---

## 3. Decorator — 같은 인터페이스로 기능 추가 (래핑)

대상과 **동일한 인터페이스**를 구현하면서, 위임 전후에 기능을 덧붙이는 패턴. Proxy와 구조가 거의 같지만 의도가 다르다 — Proxy는 **접근 제어**, Decorator는 **기능 확장**. 상속 대신 **합성으로 책임을 동적으로 쌓는다**.

```java
public interface Notifier { void send(String msg); }

public class BaseNotifier implements Notifier {
    public void send(String msg) { /* 이메일 발송 */ }
}

// 데코레이터 — 같은 인터페이스, 기능 추가 후 위임
public class SlackDecorator implements Notifier {
    private final Notifier delegate;
    public SlackDecorator(Notifier delegate) { this.delegate = delegate; }
    public void send(String msg) {
        delegate.send(msg);          // 기존 기능
        sendSlack(msg);              // 추가 기능
    }
}

// 런타임 조립: 이메일 + 슬랙 + SMS ...
Notifier n = new SmsDecorator(new SlackDecorator(new BaseNotifier()));
```

**Java I/O 스트림**이 교과서적 사례:

```java
// InputStream을 계속 감싸 기능을 누적
new BufferedReader(new InputStreamReader(new FileInputStream(file)));
//   버퍼링         바이트→문자 변환        파일 읽기
```

**Spring/실무 활용**:
- **`BeanPostProcessor`** — 빈 초기화 전후에 끼어들어 빈을 **프록시/래핑**으로 교체. AOP 프록시도 이 메커니즘으로 주입된다.
- `HttpServletRequestWrapper` / `ContentCachingRequestWrapper` — 요청을 감싸 본문 재읽기·로깅 추가.
- 캐시 데코레이터, 압축·암호화 스트림 등.

언제 쓰나:
```
✅ 기능 조합이 폭발적이라 상속 클래스 N개로 못 감당할 때 (조합 = 데코레이터 스택)
✅ 런타임에 기능을 동적으로 켜고 끄고 싶을 때
⚠️ 너무 많이 겹치면 디버깅 시 콜스택 추적이 어려워짐
```

---

## 4. Facade — 복잡 서브시스템을 단순 창구로

여러 서브시스템의 복잡한 호출 절차를 **하나의 단순한 인터페이스 뒤로 숨기는** 패턴. 클라이언트는 내부 협력 객체들을 몰라도 된다.

```java
@Service
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notification;

    public OrderResult placeOrder(OrderCommand cmd) {
        inventory.reserve(cmd.items());      // 복잡한 절차를
        payment.charge(cmd.payment());       // 한 메서드 뒤로
        shipping.schedule(cmd.address());    // 캡슐화
        notification.notifyUser(cmd.userId());
        return OrderResult.success();
    }
}
```

**Spring/실무 활용**: Spring의 **Service 계층이 사실상 Facade**다. Controller는 Repository·도메인·외부 클라이언트를 직접 다루지 않고, Service라는 단일 창구를 호출한다. Service가 여러 하위 협력자를 조율(orchestration)하고 트랜잭션 경계를 잡는다.

| 비교 | Facade | Adapter |
|------|--------|---------|
| 목적 | 복잡한 것을 **단순화** | 모양을 **변환** |
| 새 인터페이스 | 새로 만든 단순 창구 | 기존 Target에 맞춤 |

언제 쓰나:
```
✅ 서브시스템 호출 순서·의존이 복잡해 클라이언트가 알 필요 없을 때
✅ 계층 간 결합을 줄이고 진입점을 하나로 모을 때
⚠️ Facade가 모든 걸 다 하는 'God object'가 되지 않게 책임 분리 유지
```

---

## 5. Composite — 트리 구조, 부분-전체 동일 취급

객체를 **트리 구조**로 구성하고, **개별 객체(Leaf)와 복합 객체(Composite)를 같은 인터페이스로 동일하게** 다루는 패턴. 클라이언트는 단일/그룹을 구분하지 않는다.

```java
public interface MenuComponent { int price(); }      // Leaf·Composite 공통

public record MenuItem(String name, int price)       // Leaf
        implements MenuComponent {
    public int price() { return price; }
}

public class Menu implements MenuComponent {          // Composite
    private final List<MenuComponent> children = new ArrayList<>();
    public void add(MenuComponent c) { children.add(c); }
    public int price() {
        return children.stream().mapToInt(MenuComponent::price).sum(); // 재귀
    }
}
```

**Spring/실무 활용**: 메뉴/카테고리 트리, 조직도(부서-사원), 파일 시스템, `CompositeHealthContributor`(Actuator).

```
✅ 부분-전체 계층(트리)을 단일/그룹 구분 없이 처리할 때
⚠️ 평평한 컬렉션이면 과설계 — 진짜 재귀 트리일 때만
```

---

## 6. Bridge — 추상과 구현을 독립적으로 확장

**추상(abstraction)과 구현(implementation)을 별도 계층으로 분리**해 각각 독립적으로 확장하는 패턴. 두 축이 곱셈으로 늘어날 때 상속 폭발을 막는다 — 추상이 합성으로 구현을 참조한다.

```java
interface MessageSender { void send(String to, String body); }   // 구현 축
class SmsSender   implements MessageSender { ... }
class EmailSender implements MessageSender { ... }

abstract class Notification {                                     // 추상 축
    protected final MessageSender sender;   // bridge (합성)
    protected Notification(MessageSender s) { this.sender = s; }
    abstract void notify(User u);
}
// 추상 N종 × 구현 M종을 자유 조합 (상속 N×M 클래스 불필요)
```

**Adapter와의 차이**: Adapter는 **이미 존재하는** 호환 안 되는 둘을 **사후**에 끼워맞춘다. Bridge는 두 축이 독립 변화할 것을 예상하고 **사전 설계**로 분리한다.

```
✅ 두 차원(추상×구현)이 각각 독립적으로 늘어날 게 예상될 때
⚠️ 사전 설계 패턴 — 차원이 하나면 불필요한 추상화
```

---

## 7. Flyweight — 공유로 메모리 절약

동일하거나 유사한 객체가 대량으로 필요할 때, **공통 상태(intrinsic)를 공유**해 메모리를 절약하는 패턴. 변하는 상태(extrinsic)만 외부에서 주입한다.

```java
// 풀에서 공유 객체 재사용
public class FontFlyweightFactory {
    private static final Map<String, Font> pool = new ConcurrentHashMap<>();
    public static Font of(String name) {
        return pool.computeIfAbsent(name, Font::new); // 같은 폰트는 재사용
    }
}
```

**Java 기본 라이브러리 사례**:
- **String pool** — 동일 문자열 리터럴은 하나의 인스턴스를 공유 (`"a" == "a"`).
- **`Integer.valueOf` 캐시** — -128~127 범위는 캐시된 인스턴스 반환 (`Integer.valueOf(100) == Integer.valueOf(100)` → true, 128은 false).
- `Boolean`, `Character` 등도 동일.

언제 쓰나:
```
✅ 동일 객체가 대량 생성돼 메모리 압박이 큰데, 불변 공통 상태로 공유 가능할 때
⚠️ 공유 객체는 반드시 불변(immutable) — 가변이면 공유가 버그의 원천
```

---

## 8. 7개 한눈에 비교

| 패턴 | 한 줄 의도 | 인터페이스 | Spring 대표 활용 |
|------|-----------|-----------|------------------|
| **Adapter** | 모양 변환 | 바꿈 | 외부 SDK Port, HandlerAdapter |
| **Proxy** | 접근 제어/부가 행위 | 동일 | **AOP @Transactional**, JPA 지연 로딩 |
| **Decorator** | 기능 추가 | 동일 | BeanPostProcessor, I/O 스트림 |
| **Facade** | 복잡함 단순화 | 새 창구 | **Service 계층** |
| **Composite** | 부분-전체 트리 | 동일 | 메뉴/조직도 트리 |
| **Bridge** | 추상-구현 분리 | 분리(사전) | 추상×구현 독립 확장 |
| **Flyweight** | 공유로 절약 | - | String pool, Integer 캐시 |

> 헷갈림 포인트: Proxy·Decorator는 구조가 같다 — **의도**로 구분(제어 vs 확장). Adapter vs Bridge는 **시점**으로 구분(사후 vs 사전). Adapter vs Facade는 **목적**으로 구분(변환 vs 단순화).

---

## 관련
- [[Adapter-Pattern]] — Adapter 상세 (Hexagonal Ports & Adapters)
- [[Spring-Core]] — DI·AOP·BeanPostProcessor의 기반
- [[Creational-Patterns]] — 생성 패턴 (객체 생성 방법)
- [[Behavioral-Patterns]] — 행위 패턴 (객체 간 책임 분배)
