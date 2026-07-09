---
tags:
  - pattern
  - design-pattern
  - behavioral
  - gof
created: 2026-06-17
---

# 행위 패턴 (Behavioral Patterns) — GoF

> [!summary] 한 줄 요약
> 객체·클래스 간 **책임 분배와 상호작용(알고리즘·흐름·통신)** 을 다루는 GoF 패턴군. 실무에서는 대부분 Spring이 프레임워크 차원에서 구현해 둔 형태(DI 전략 주입, `JdbcTemplate`, `ApplicationEvent`, `Filter` 체인)로 만난다. 직접 구현보다 **"어디에 이미 녹아 있는지"** 를 아는 것이 핵심.

---

## 1. Strategy — 알고리즘 캡슐화·교체 ⭐

알고리즘 군(群)을 각각 클래스로 캡슐화하고, 런타임에 **교체 가능**하게 만든다. `if/switch`로 분기하던 동작을 다형성으로 치환해 OCP를 확보한다.

```java
public interface DiscountPolicy {
    Money discount(Order order);
}

@Component("rate")
class RateDiscount implements DiscountPolicy {
    public Money discount(Order o) { return o.total().times(0.1); }
}

@Component("fixed")
class FixedDiscount implements DiscountPolicy {
    public Money discount(Order o) { return Money.won(3000); }
}
```

### Spring/실무 활용 — DI로 전략 주입

Spring에서 Strategy는 **인터페이스 + 여러 구현 빈**으로 자연스럽게 표현된다. 주입 방식이 곧 전략 선택 메커니즘.

```java
// (1) @Qualifier 로 특정 전략 선택
@Service
@RequiredArgsConstructor
class CheckoutService {
    @Qualifier("rate")
    private final DiscountPolicy policy;   // 컴파일 타임 고정
}

// (2) Map 주입 — 빈 이름이 key, 런타임 동적 선택
@Service
@RequiredArgsConstructor
class DiscountSelector {
    private final Map<String, DiscountPolicy> policies;  // {"rate":.., "fixed":..}

    Money apply(String type, Order order) {
        return policies.getOrDefault(type, policies.get("fixed"))
                       .discount(order);
    }
}

// (3) List 주입 + 자기 책임 판별 (체인 비슷한 디스패치)
@Service
@RequiredArgsConstructor
class PaymentRouter {
    private final List<PaymentProcessor> processors;
    void pay(PayRequest r) {
        processors.stream().filter(p -> p.supports(r.method()))
                  .findFirst().orElseThrow().process(r);
    }
}
```

> Spring이 같은 타입 빈 여러 개를 `Map<String, T>`/`List<T>`로 자동 수집하는 것이 Strategy 주입의 핵심. `@Qualifier`는 정적, Map은 동적 선택에 쓴다.

**언제 쓰나**: 조건 분기로 동작이 갈리고 그 종류가 늘어날 때, A/B 정책·결제수단·할인정책처럼 **교체 단위가 명확**할 때.

---

## 2. Template Method — 골격 정의 + 세부는 위임 ⭐

상위에서 알고리즘 **골격(순서)** 을 고정하고, 변하는 단계만 서브클래스(또는 콜백)에 위임한다. "흐름은 내가, 빈칸은 너가."

```java
abstract class ReportGenerator {
    public final Report generate() {     // 골격은 final로 고정
        var data = fetch();
        var body = render(data);          // 가변 단계
        return new Report(header(), body);
    }
    protected String header() { return "기본 헤더"; }  // 훅(기본 구현)
    protected abstract RawData fetch();                // 강제 구현
    protected abstract String render(RawData d);
}
```

### Spring/실무 활용 — `*Template` 군

Spring의 `JdbcTemplate`, `RestTemplate`, `TransactionTemplate`, `RedisTemplate`은 모두 Template Method의 **콜백 변형**이다. 연결 획득→실행→예외변환→자원 해제라는 **반복 골격을 프레임워크가 쥐고**, 사용자는 SQL 매핑 같은 가변부만 람다로 제공한다.

```java
List<User> users = jdbcTemplate.query(
    "SELECT id, name FROM users WHERE age > ?",
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")),  // 가변부만
    18);
// 커넥션 open/close, 예외→DataAccessException 변환은 템플릿이 처리
```

> 고전 Template Method는 **상속**으로, Spring `*Template`은 **콜백(전략) 위임**으로 변하는 부분을 주입한다. 후자가 결합도가 낮아 실무 기본형. 상속형은 `WebSecurityConfigurerAdapter`(구버전) 같은 Abstract base에서 보였다.

**언제 쓰나**: 처리 순서는 동일한데 일부 단계만 다를 때, 자원 관리(open/close)·예외 변환 같은 **보일러플레이트를 봉인**하고 싶을 때.

---

## 3. Observer — 상태 변화 통지 ⭐

Subject의 상태가 바뀌면 등록된 Observer들에게 **자동 통지(발행/구독)** 한다. 발신자는 수신자가 누구인지 몰라 단방향으로 느슨하게 결합된다.

```java
interface Observer { void onChanged(Event e); }

class Subject {
    private final List<Observer> observers = new CopyOnWriteArrayList<>();
    void subscribe(Observer o) { observers.add(o); }
    void publish(Event e) { observers.forEach(o -> o.onChanged(e)); }
}
```

### Spring/실무 활용 — `ApplicationEvent` / `@EventListener`

Spring은 컨테이너 차원의 Observer를 내장한다. `ApplicationEventPublisher`가 Subject, `@EventListener` 빈들이 Observer.

```java
// 발행 — 누가 듣는지 모름
@Service
@RequiredArgsConstructor
class OrderService {
    private final ApplicationEventPublisher publisher;
    @Transactional
    public void place(Order o) {
        orderRepository.save(o);
        publisher.publishEvent(new OrderPlacedEvent(o.getId()));  // 통지
    }
}

// 구독 — 독립적으로 반응 (메일, 재고, 통계 …)
@Component
class MailNotifier {
    @TransactionalEventListener(phase = AFTER_COMMIT)   // 커밋 후에만
    @Async
    void on(OrderPlacedEvent e) { mailSender.sendConfirm(e.orderId()); }
}
```

> `@TransactionalEventListener(AFTER_COMMIT)`로 **트랜잭션 커밋 후** 처리해 "DB 롤백됐는데 메일 발송" 같은 사고를 막는다. 비동기는 `@Async` + `@EnableAsync`. 단, 인메모리 이벤트라 프로세스 죽으면 유실 → 신뢰성 필요 시 [[Outbox-Pattern]].

**언제 쓰나**: 한 사건에 **여러 부수 효과**가 붙고 그 목록이 늘어날 때, 도메인 핵심 로직과 부가 로직(알림·감사·통계)을 분리하고 싶을 때.

---

## 4. Command — 요청을 객체로 캡슐화

요청(수신자·메서드·인자)을 하나의 객체로 감싼다. 요청을 **큐에 쌓고, 로깅하고, 실행 취소(undo)** 할 수 있게 된다.

```java
interface Command { void execute(); void undo(); }

class AddTextCommand implements Command {
    private final Document doc; private final String text;
    AddTextCommand(Document doc, String text) { this.doc = doc; this.text = text; }
    public void execute() { doc.append(text); }
    public void undo()    { doc.removeLast(text.length()); }
}

class CommandStack {                       // invoker
    private final Deque<Command> history = new ArrayDeque<>();
    void run(Command c) { c.execute(); history.push(c); }
    void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```

**실무**: 작업 큐(메시지 핸들러), 트랜잭션/배치 작업의 재실행, 매크로·undo. CQRS의 `Command` 객체도 이 계열(→ [[Mediator-Pattern]], [[CQRS]]). Spring `@Async` 작업, `Runnable`/`Callable` 캡슐화도 변형.

**언제 쓰나**: 요청을 **데이터처럼 다뤄야** 할 때 — 지연 실행, 큐잉, 이력/undo, 트랜잭션 단위 묶음.

---

## 5. Chain of Responsibility — 처리 객체 체인

요청을 처리할 수 있는 핸들러들을 **사슬로 연결**하고, 각 핸들러가 처리하거나 다음으로 넘긴다. 발신자는 어느 핸들러가 처리할지 모른다.

```java
abstract class Handler {
    protected Handler next;
    Handler then(Handler n) { this.next = n; return n; }
    abstract void handle(Request r);
    protected void pass(Request r) { if (next != null) next.handle(r); }
}
```

### Spring/실무 활용 — Servlet Filter / HandlerInterceptor

서블릿 `Filter` 체인이 교과서적 CoR이다. 각 필터가 `doFilter(req, res, chain)`에서 처리 후 `chain.doFilter(...)`로 다음에 넘긴다.

```java
@Component
class AuthFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        if (!authenticated(req)) { reject(res); return; }  // 체인 중단
        chain.doFilter(req, res);                          // 다음 필터로
    }
}

// Spring MVC 계층의 HandlerInterceptor — preHandle 이 false면 체인 중단
@Component
class LoggingInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest r, HttpServletResponse s, Object h) {
        log.info("{} {}", r.getMethod(), r.getRequestURI());
        return true;   // false 반환 시 이후 인터셉터/핸들러 미실행
    }
}
```

> Spring Security는 `SecurityFilterChain`으로 인증·인가·CSRF 필터를 줄세운 거대한 CoR다. 각 단계가 "내 책임이면 처리, 아니면 통과"를 반복.

**언제 쓰나**: 요청을 **여러 단계로 순차 가공/검증**할 때, 처리 단계를 동적으로 끼우고 빼고 싶을 때(미들웨어).

---

## 6. State — 상태별 행위 캡슐화

객체의 **내부 상태**에 따라 행위가 바뀔 때, 각 상태를 클래스로 분리해 `if (state == ...)` 분기를 없앤다. 상태 전이 규칙이 코드에 명시된다.

```java
interface OrderState {
    OrderState pay(Order o);
    OrderState cancel(Order o);
}

class Created implements OrderState {
    public OrderState pay(Order o)    { o.charge(); return new Paid(); }
    public OrderState cancel(Order o) { return new Canceled(); }
}
class Paid implements OrderState {
    public OrderState pay(Order o)    { throw new IllegalStateException("이미 결제됨"); }
    public OrderState cancel(Order o) { o.refund(); return new Canceled(); }
}
```

**실무**: 주문/결제/배송 **상태 머신**, 워크플로 엔진(Spring StateMachine). 상태가 많고 전이 규칙이 복잡하면 enum 분기보다 State 객체나 전용 상태머신이 유지보수에 유리. 분산 환경의 장기 트랜잭션 상태 전이는 [[Saga-Pattern]]이 오케스트레이션한다.

**언제 쓰나**: 동일 메서드가 상태에 따라 전혀 다르게 동작하고, **허용/금지 전이**를 강제하고 싶을 때.

---

## 7. Mediator — 중앙 중재자

객체들이 서로 직접 참조하지 않고 **중앙의 Mediator**를 통해서만 통신해, N:N 결합을 N:1로 낮춘다. CQRS에서는 Command/Query를 Handler로 **디스패치**하는 축으로 확장된다.

> 상세 구조·Spring 핸들러 맵 디스패치·Observer와의 차이는 [[Mediator-Pattern]] 참고.

**언제 쓰나**: 객체 간 의존이 그물처럼 얽혀 변경이 파급될 때, 통신 규칙을 한 곳에서 관리하고 싶을 때.

---

## 8. 기타 — Iterator / Visitor / Memento / Interpreter

| 패턴 | 개념 | 예 |
|------|------|-----|
| **Iterator** | 내부 표현을 노출하지 않고 컬렉션 요소를 **순차 접근**. | Java `Iterator`/`Iterable`, `for-each`, `Stream`. 사실상 언어에 내장됨. |
| **Visitor** | 자료 구조와 **연산을 분리** — 구조 변경 없이 새 연산 추가. 데이터 클래스는 `accept(visitor)`만. | AST 순회(컴파일러), 파일 트리 처리(`FileVisitor`), JPA Criteria 빌더 내부. 타입별 처리(`sealed` + switch 패턴 매칭)로 대체되는 추세. |
| **Memento** | 캡슐화를 깨지 않고 객체의 **상태 스냅샷**을 저장·복원. | undo/redo, 트랜잭션 세이브포인트, 게임 세이브. |
| **Interpreter** | 언어의 문법을 클래스로 표현해 **문장을 해석**. | 표현식 엔진(SpEL), 규칙 엔진, 쿼리 파서. 직접 구현은 드물고 라이브러리로 대체. |

> 이 4개는 실무에서 직접 구현할 일이 적다. Iterator·Visitor는 언어/프레임워크에 흡수됐고, Interpreter는 SpEL·정규식·파서 라이브러리로 대체된다. 개념과 "어디 녹아 있는지"만 알면 충분.

---

## 정리 — 실무 매핑 한눈에

| 패턴 | Spring/Java에서의 화신 | 핵심 키워드 |
|------|------------------------|-------------|
| Strategy | DI 인터페이스 다구현, `Map`/`List` 주입, `@Qualifier` | 알고리즘 교체 |
| Template Method | `JdbcTemplate`·`RestTemplate`·`TransactionTemplate` | 골격 고정·콜백 |
| Observer | `ApplicationEventPublisher` + `@EventListener` | 발행/구독 |
| Command | CQRS Command, `Runnable`/`Callable`, 작업 큐 | 요청 객체화 |
| CoR | Servlet `Filter`, `HandlerInterceptor`, Security FilterChain | 핸들러 체인 |
| State | 상태 머신, Spring StateMachine | 상태별 행위 |
| Mediator | 핸들러 맵 디스패처, `ApplicationEventPublisher` | 중앙 통신 |

---

## 관련
- [[Mediator-Pattern]] — 행위 패턴 중 중재자의 심화(CQRS 디스패치)
- [[Spring-Core]] — DI/이벤트 발행이 Strategy·Observer의 토대
- [[Creational-Patterns]] — 객체 생성 패턴(GoF)
- [[Structural-Patterns]] — 객체 구조 패턴(GoF)
- [[Saga-Pattern]] — State의 분산 확장(장기 트랜잭션 상태 전이)

## 참고
- GoF, *Design Patterns* — Behavioral Patterns
- refactoring.guru — Behavioral Patterns
- Spring Framework Reference — Events, JdbcTemplate, Filter/Interceptor
