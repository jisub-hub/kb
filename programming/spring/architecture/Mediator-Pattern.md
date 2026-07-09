---
tags:
  - pattern
  - design-pattern
  - behavioral
  - cqrs
  - mediator
created: 2026-06-17
---

# Mediator Pattern (중재자 패턴)

> [!summary] 한 줄 요약
> 객체들이 **서로 직접 참조하지 않고**, 중앙의 **중재자(Mediator)** 를 통해서만 통신하게 하여 객체 간 결합도를 낮추는 행위(behavioral) 패턴. CQRS에서는 Command/Query를 **Handler로 디스패치**하는 축으로 확장된다.

---

## 1. 개념 (GoF Mediator)

여러 객체가 서로 직접 참조하면 관계가 **N:N 그물(mesh)** 로 얽혀, 한 객체의 변경이 다수에 파급된다. Mediator는 이 통신을 **중앙으로 모은다**.

```
[직접 참조: N:N 그물]            [Mediator: N:1 스타]
  A ─ B                            A     B
  │ ╳ │            ───►             ╲   ╱
  C ─ D                          ┌── Mediator ──┐
                                  C            D
```

- 각 객체(Colleague)는 다른 객체를 모르고 **Mediator만 안다**.
- 상호작용 로직이 Mediator에 **집중**되어, 객체 간 결합이 N:N → N:1로 줄어든다.

---

## 2. 언제 쓰나

- 객체 간 의존이 복잡하게 얽혀 **재사용·변경이 어려울 때**.
- 한 객체의 동작이 여러 객체에 영향을 줘 동작이 흩어져 있을 때.
- 통신 규칙을 **한 곳에서 관리·변경**하고 싶을 때 (예: 폼의 위젯 간 활성/비활성 규칙).
- [[Spring-Core]]에서 빈 간 **순환 의존성**을 끊고 싶을 때 (직접 참조 제거).

---

## 3. 구조 (Mediator / Colleague)

```java
// Mediator: Colleague 간 상호작용을 중재
interface DialogMediator {
    void notify(Component sender, String event);
}

abstract class Component {
    protected final DialogMediator mediator;
    protected Component(DialogMediator mediator) { this.mediator = mediator; }
}

class Checkbox extends Component {
    Checkbox(DialogMediator m) { super(m); }
    void toggle() { mediator.notify(this, "toggle"); }  // 다른 위젯을 직접 모름
}

class SubmitButton extends Component {
    SubmitButton(DialogMediator m) { super(m); }
    void enable() { /* ... */ }
}

class SignupDialog implements DialogMediator {
    private Checkbox agree;
    private SubmitButton submit;
    public void notify(Component sender, String event) {
        if (sender == agree && event.equals("toggle")) {
            submit.enable();        // 상호작용 규칙이 여기에 집중
        }
    }
}
```

- **Mediator**: 중재 인터페이스. **ConcreteMediator**: 상호작용 규칙 구현.
- **Colleague**: Mediator를 통해서만 통신하는 참여 객체.

---

## 4. CQRS와의 연결 — Handler 디스패치

[[CQRS]]에서 Mediator는 **요청(Command/Query) 객체를 그에 맞는 Handler로 보내주는 디스패처**로 변형되어 쓰인다. 발신자(Controller)는 어떤 Handler가 처리하는지 **몰라도 된다**.

```
Controller ──► Mediator.send(command) ──► (타입 매칭) ──► PlaceOrderHandler
                    (디스패처)
```

- .NET 진영의 **MediatR** 라이브러리가 이 형태를 대중화: `_mediator.Send(command)` 한 줄로 핸들러를 찾아 실행.
- 발신과 처리가 분리되어, 핸들러 추가 시 Controller를 **수정하지 않는다** (OCP).
- 파이프라인(검증·로깅·트랜잭션)을 데코레이터처럼 끼우기 좋다.

> Java에는 표준 MediatR가 없다. 보통 ① Spring `ApplicationEventPublisher`(이벤트형) ② 직접 핸들러 맵 디스패치 ③ Axon `CommandBus`/`QueryBus` 중 택한다.

---

## 5. Spring에서의 구현 — 핸들러 맵 디스패치

Spring DI를 이용하면 핸들러들을 자동 수집해 **타입 → 핸들러** 맵을 만들고 디스패치할 수 있다.

```java
// 1) 마커 인터페이스
public interface Command<R> {}

public interface CommandHandler<C extends Command<R>, R> {
    R handle(C command);
    Class<C> commandType();      // 자신이 처리할 Command 타입
}

// 2) Mediator: 주입받은 핸들러들로 타입→핸들러 맵 구성
@Component
public class Mediator {
    private final Map<Class<?>, CommandHandler<?, ?>> handlers = new HashMap<>();

    public Mediator(List<CommandHandler<?, ?>> handlerBeans) {  // Spring이 모두 주입
        handlerBeans.forEach(h -> handlers.put(h.commandType(), h));
    }

    @SuppressWarnings("unchecked")
    public <R> R send(Command<R> command) {
        var handler = (CommandHandler<Command<R>, R>) handlers.get(command.getClass());
        if (handler == null)
            throw new IllegalStateException("핸들러 없음: " + command.getClass());
        return handler.handle(command);
    }
}

// 3) Command + Handler
public record PlaceOrderCommand(String customerId, List<OrderItem> items)
        implements Command<UUID> {}

@Component
@RequiredArgsConstructor
public class PlaceOrderHandler implements CommandHandler<PlaceOrderCommand, UUID> {
    private final OrderRepository orderRepository;

    @Override public UUID handle(PlaceOrderCommand cmd) {
        Order order = Order.place(cmd.customerId(), cmd.items());
        orderRepository.save(order);
        return order.getId();
    }
    @Override public Class<PlaceOrderCommand> commandType() { return PlaceOrderCommand.class; }
}

// 4) Controller — 어떤 핸들러가 처리하는지 모름
@RestController
@RequiredArgsConstructor
class OrderController {
    private final Mediator mediator;

    @PostMapping("/orders")
    UUID place(@RequestBody PlaceOrderCommand cmd) {
        return mediator.send(cmd);   // 디스패치는 Mediator가 책임
    }
}
```

> Query 측도 동일 구조(`Query<R>` / `QueryHandler`)로 둔다. 검증·로깅·트랜잭션은 `send()` 앞뒤에 파이프라인(데코레이터)으로 추가한다.

> [!tip] 이벤트형 대안
> 반환값이 필요 없는 발행/구독이면 `ApplicationEventPublisher.publishEvent(...)` + `@EventListener` 가 더 가볍다. **반환값을 받는 명령 디스패치**가 필요할 때 위의 직접 구현이 유용하다.

---

## 6. 장점 / 단점

### ✅ 장점
- **결합도 감소**: Colleague 간 직접 참조 제거 (N:N → N:1).
- **상호작용 로직 집중**: 통신 규칙을 한 곳에서 관리·테스트.
- **확장 용이**: 새 핸들러/Colleague 추가 시 발신 측 코드 변경 없음(OCP).
- **순환 의존성 회피**: 빈이 서로를 직접 참조하지 않음 → [[Spring-Core]] 9절.

### ❌ 단점
- **God Object 위험**: Mediator에 로직이 과도하게 몰리면 그 자체가 복잡한 거대 객체가 된다.
- **간접성 증가**: 호출 흐름 추적이 어려워질 수 있다(디버깅 시 한 단계 우회).
- **과용 주의**: 단순한 직접 호출로 충분한 곳에 도입하면 오버엔지니어링.

---

## 7. Observer 패턴과의 차이

| | **Mediator** | **Observer** |
|--|--------------|--------------|
| 목적 | 객체 간 **상호 통신**을 중앙 집중 | 상태 변화의 **단방향 알림(발행/구독)** |
| 관계 | 다대다 통신을 N:1로 정리 | Subject 1 → 다수 Observer |
| 결합 방향 | Colleague ↔ Mediator (양방향 조율) | Subject → Observer (한 방향 통지) |
| 비유 | 관제탑(쌍방 교신 조율) | 신문 구독(발행하면 구독자에 전달) |

> 둘은 배타적이지 않다. Mediator 내부에서 Observer로 변경을 통지하도록 결합하기도 한다. Spring의 `ApplicationEventPublisher` 는 Observer에 가깝고, 4~5절의 Command 디스패처는 Mediator에 가깝다.

---

## 관련
- [[CQRS]] — Command/Query 디스패치에 Mediator 적용
- [[Spring-Core]] — DI/이벤트 발행, 순환 의존성 회피
- [[Saga-Pattern]] — 오케스트레이터도 일종의 단계 중재자
- [[Event-Sourcing]] — 이벤트 기반 통신과 결합

## 참고
- GoF, *Design Patterns* — Mediator
- Jimmy Bogard, **MediatR** (.NET) — CQRS 디스패치 대중화
- refactoring.guru — Mediator
