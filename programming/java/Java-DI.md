---
tags:
  - java
  - di
  - ioc
  - spring
  - design-patterns
created: 2026-06-15
---

# Dependency Injection (의존성 주입)

> 객체가 필요한 의존성을 스스로 생성하지 않고 **외부에서 주입**받는 설계 원칙. SOLID의 DIP를 실현하는 메커니즘.

---

## IoC vs DI

| 개념 | 설명 |
|------|------|
| **IoC** (Inversion of Control) | 프로그램 흐름 제어권을 프레임워크에게 역전 |
| **DI** (Dependency Injection) | IoC를 구현하는 패턴 — 의존성을 외부에서 주입 |
| **IoC Container** | DI를 자동화하는 컨테이너 (Spring의 `ApplicationContext`) |

---

## DI 3가지 방식

### 1. 생성자 주입 (Constructor Injection) ✅ 권장
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    // Spring 4.3+ 단일 생성자면 @Autowired 생략 가능
    @Autowired
    public OrderService(OrderRepository orderRepository,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }
}

// Lombok @RequiredArgsConstructor 로 축약
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final NotificationService notificationService;
}
```

### 2. Setter 주입 (Setter Injection)
```java
@Service
public class OrderService {
    private OrderRepository orderRepository;

    @Autowired
    public void setOrderRepository(OrderRepository repo) {
        this.orderRepository = repo;
    }
}
```

### 3. 필드 주입 (Field Injection) ❌ 비권장
```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // 테스트·불변성 문제
}
```

---

## 왜 생성자 주입인가

| 기준 | 생성자 | Setter | 필드 |
|------|--------|--------|------|
| **불변성** | `final` 선언 가능 ✅ | 불가 ❌ | 불가 ❌ |
| **테스트** | `new` 로 직접 주입 가능 ✅ | 가능 ✅ | Reflection 필요 ❌ |
| **순환의존 감지** | 시작 시 실패 ✅ | 런타임 실패 ❌ | 런타임 실패 ❌ |
| **필수 의존성 명시** | 명확 ✅ | 불명확 ❌ | 숨겨짐 ❌ |

```java
// 생성자 주입 → 테스트 코드에서 Mock 주입이 쉬움
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @Mock NotificationService notifier;

    OrderService service;

    @BeforeEach
    void setUp() {
        service = new OrderService(repo, notifier);  // DI 컨테이너 없이 테스트 가능
    }
}
```

---

## Spring IoC Container

```
@Component / @Service / @Repository / @Controller
         ↓  컴포넌트 스캔
  ApplicationContext (BeanFactory 확장)
         ↓  Bean 등록·관리
  의존성 그래프 해결 → 주입
```

### Bean 등록 방법
```java
// 1. 어노테이션 기반 (컴포넌트 스캔)
@Component public class MyBean { ... }
@Service   public class MyService { ... }    // @Component + 서비스 계층 의미
@Repository public class MyRepo { ... }     // @Component + 예외 변환
@Controller public class MyCtrl { ... }     // @Component + MVC

// 2. @Configuration + @Bean (외부 라이브러리, 세밀한 제어)
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(hikariConfig());
    }
}
```

---

## 의존성 해결 우선순위

같은 타입 Bean이 여러 개일 때:

```java
interface PaymentStrategy { ... }

@Component @Primary          // 기본값으로 주입
class CardPayment implements PaymentStrategy { ... }

@Component
class KakaoPayment implements PaymentStrategy { ... }

// 수신 측
@Service
@RequiredArgsConstructor
class PaymentService {
    private final PaymentStrategy payment;  // → CardPayment 주입
}
```

```java
// @Qualifier — 이름으로 명시
@Service
public class PaymentService {
    private final PaymentStrategy payment;

    public PaymentService(@Qualifier("kakaoPayment") PaymentStrategy payment) {
        this.payment = payment;
    }
}
```

```java
// 여러 구현체 전부 받기
@Service
@RequiredArgsConstructor
class PaymentRouter {
    private final List<PaymentStrategy> strategies;  // 모든 구현체 주입
    private final Map<String, PaymentStrategy> strategyMap;  // 이름으로 맵핑
}
```

---

## Bean 스코프

| 스코프 | 설명 | 사용처 |
|--------|------|--------|
| `singleton` | 기본. 컨테이너당 1개 | 대부분 |
| `prototype` | 요청마다 새 인스턴스 | 상태 있는 Bean |
| `request` | HTTP 요청당 1개 | Web MVC |
| `session` | HTTP 세션당 1개 | Web MVC |

```java
@Component
@Scope("prototype")
public class StatefulBean { ... }
```

---

## 순환 의존성 (Circular Dependency)

```
A → B → A  (생성자 주입 시 시작 시점에 BeanCurrentlyInCreationException)
```

**해결 우선순위**
1. 설계 재검토 — 순환은 대개 책임 분리 실패 신호
2. 중간 객체(이벤트·인터페이스) 도입으로 의존 방향 정리
3. `@Lazy` — 실제 사용 시점까지 초기화 지연 (최후 수단)

```java
@Service
@RequiredArgsConstructor
public class A {
    @Lazy
    private final B b;
}
```

---

## 관련
- [[OOP-SOLID]] (DIP) · [[Java-AOP]] · [[Java-Patterns]] (Dynamic Proxy)
