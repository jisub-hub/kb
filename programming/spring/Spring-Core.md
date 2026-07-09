---
tags:
  - spring
  - core
  - ioc
  - di
  - bean
created: 2026-06-17
---

# Spring Core (IoC / DI 기초)

> [!summary] 한 줄 요약
> 객체의 **생성·생명주기·의존성 연결**을 개발자 대신 **컨테이너(IoC Container)** 가 책임지게 하고, 그 의존성을 **외부에서 주입(DI)** 하는 것이 Spring의 핵심.

---

## 1. IoC (제어의 역전)

**제어의 역전(Inversion of Control)** 은 객체의 생성과 의존 관계 연결의 **제어권을 개발자에서 프레임워크(컨테이너)로 넘기는 것**이다.

```java
// ❌ 전통적 방식: 객체가 자신의 의존성을 직접 생성·제어
class OrderService {
    private OrderRepository repo = new MySQLOrderRepository(); // 강결합
}

// ✅ IoC: 누가 무엇을 주입할지는 컨테이너가 결정
@Service
class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo) { this.repo = repo; } // 받아 쓰기만
}
```

- "내가 객체를 만든다"가 아니라 **"컨테이너가 만들어 넣어준다"** 로 흐름이 뒤집힌다.
- 이는 [[OOP-SOLID]]의 **DIP(의존성 역전 원칙)** 를 프레임워크 차원에서 실현한 것.

> IoC는 개념, **DI는 그 IoC를 구현하는 가장 대표적인 방식**이다.

---

## 2. DI (의존성 주입) — 3가지 방식

| 방식 | 형태 | 평가 |
|------|------|------|
| **생성자 주입** | 생성자 파라미터 | ✅ **권장** |
| **세터 주입** | setter 메서드 | 선택적/변경 가능한 의존성에만 |
| **필드 주입** | 필드에 `@Autowired` | ❌ 지양 (테스트·불변성 불리) |

```java
@Service
@RequiredArgsConstructor   // Lombok: final 필드 생성자 자동 생성
class OrderService {
    private final OrderRepository repo;        // 생성자 주입
    private final PaymentClient paymentClient;
}
```

### 왜 생성자 주입인가
- **불변성**: 필드를 `final` 로 선언 → 주입 후 변경 불가, 안전.
- **필수 의존성 명확화**: 객체 생성 시점에 의존성이 모두 채워짐을 보장 (NPE 예방).
- **테스트 용이**: 컨테이너 없이 `new OrderService(mockRepo, ...)` 로 주입 가능.
- **순환 의존성 조기 발견**: 애플리케이션 기동 시점에 실패 → 런타임이 아닌 부팅 단계에서 잡힘.

> 생성자가 1개면 `@Autowired` 생략 가능. Lombok `@RequiredArgsConstructor` 가 사실상 표준.

---

## 3. Bean · Container · ApplicationContext

- **Bean**: 컨테이너가 생성·관리하는 객체. 기본은 **싱글톤**.
- **BeanFactory**: 가장 기본적인 IoC 컨테이너 인터페이스. 지연 로딩(lazy).
- **ApplicationContext**: `BeanFactory` 를 확장한 상위 인터페이스. **실무에서 쓰는 것은 거의 이것.**

| 구분 | BeanFactory | ApplicationContext |
|------|-------------|--------------------|
| Bean 로딩 | 요청 시 (lazy) | 기동 시 (eager, 싱글톤) |
| 국제화(i18n) | ✗ | ✓ |
| 이벤트 발행 | ✗ | ✓ (`ApplicationEventPublisher`) |
| AOP/애너테이션 | 제한적 | 완전 지원 |

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService service = ctx.getBean(OrderService.class);
```

---

## 4. Bean 스코프

| 스코프 | 설명 |
|--------|------|
| **singleton** (기본) | 컨테이너당 인스턴스 1개. 상태 없는(stateless) 빈에 적합 |
| **prototype** | 요청(주입/조회)마다 새 인스턴스. 컨테이너는 생성까지만 관리 |
| **request** | HTTP 요청당 1개 (웹 한정) |
| **session** | HTTP 세션당 1개 (웹 한정) |
| **application** | ServletContext당 1개 |

```java
@Component
@Scope("prototype")
class TaskWorker { /* 호출마다 새 인스턴스 */ }
```

> [!warning] 싱글톤 빈의 상태 공유
> 싱글톤 빈은 모든 요청이 **공유**한다. 가변 인스턴스 필드를 두면 동시성 버그가 발생한다. 빈은 **무상태**로 설계할 것.

> [!tip] 싱글톤 안에 prototype 주입
> 싱글톤에 prototype을 그냥 주입하면 **한 번만** 주입되어 prototype 효과가 사라진다. → `ObjectProvider` / `@Lookup` / Provider 패턴으로 매번 새로 받아온다.

---

## 5. Bean 생명주기

```
인스턴스화 → 의존성 주입 → @PostConstruct → (사용) → @PreDestroy → 소멸
```

```java
@Component
class CacheLoader {
    @PostConstruct
    void init() { /* 의존성 주입 완료 후 초기화 (예: 캐시 워밍업) */ }

    @PreDestroy
    void cleanup() { /* 컨테이너 종료 직전 자원 정리 (예: 커넥션 close) */ }
}
```

- `@PostConstruct` / `@PreDestroy` (JSR-250) 가 가장 권장되는 콜백.
- 대안: `InitializingBean`/`DisposableBean` 인터페이스, `@Bean(initMethod=, destroyMethod=)`.
- prototype 빈은 **소멸 콜백이 호출되지 않는다** (컨테이너가 생성 후 관리 종료).

---

## 6. 빈 등록 — @Component / @Bean / @Configuration

```java
// 방식 A) @Component 계열 + 컴포넌트 스캔 (내가 만든 클래스)
@Service          // @Component의 의미 특화 버전: @Repository, @Controller
class OrderService { ... }

// 방식 B) @Bean (외부 라이브러리 객체 등 직접 제어가 필요한 경우)
@Configuration
class AppConfig {
    @Bean
    ObjectMapper objectMapper() {
        return new ObjectMapper().registerModule(new JavaTimeModule());
    }
}
```

| 구분 | `@Component` | `@Bean` |
|------|--------------|---------|
| 대상 | 직접 작성한 클래스 | 외부/제3자 클래스, 세밀한 생성 제어 |
| 등록 단위 | 클래스 | 메서드 반환값 |
| 발견 방식 | 컴포넌트 스캔 | `@Configuration` 내 선언 |

> `@Configuration` 클래스의 `@Bean` 메서드는 **CGLIB 프록시**로 감싸져, 메서드를 여러 번 호출해도 같은 싱글톤을 반환한다 (`proxyBeanMethods=true` 기본).

---

## 7. 컴포넌트 스캔

`@SpringBootApplication` (= `@ComponentScan` 포함) 이 위치한 패키지를 **루트로 하위 패키지를 스캔**하여 `@Component` 계열 빈을 자동 등록한다.

```java
@SpringBootApplication   // 보통 최상위 패키지에 둔다
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

> 메인 클래스를 최상위 패키지에 두는 것이 관례. 빠뜨린 패키지의 빈은 등록되지 않는다.

---

## 8. @Autowired와 한정자 (@Qualifier / @Primary)

같은 타입의 빈이 **둘 이상**이면 어느 것을 주입할지 모호해진다(`NoUniqueBeanDefinitionException`).

```java
interface PaymentClient {}

@Component @Primary           // 기본 후보로 우선 선택
class CardPaymentClient implements PaymentClient {}

@Component
class KakaoPaymentClient implements PaymentClient {}

@Service
@RequiredArgsConstructor
class CheckoutService {
    @Qualifier("kakaoPaymentClient")          // 명시적으로 특정 빈 지목
    private final PaymentClient paymentClient;
}
```

- **`@Primary`**: 후보 중 우선순위. 기본값 성격.
- **`@Qualifier`**: 이름으로 정확히 지목. `@Primary` 보다 우선한다.
- 여러 구현을 한꺼번에 받고 싶으면 `List<PaymentClient>` / `Map<String, PaymentClient>` 로 주입.

---

## 9. 순환 의존성 (Circular Dependency)

A가 B를, B가 A를 생성자 주입으로 요구하면 컨테이너가 둘 다 만들 수 없어 **기동 시 실패**한다.

```
A ──생성자 주입──► B
▲                  │
└────생성자 주입────┘   → BeanCurrentlyInCreationException
```

| 해결책 | 설명 |
|--------|------|
| **설계 재검토** | 대부분 책임 분리 신호. 공통 로직을 제3의 빈으로 추출 (근본 해결) |
| **이벤트 기반 분리** | `ApplicationEventPublisher` 로 직접 참조 제거 → [[Mediator-Pattern]] |
| `@Lazy` | 한쪽을 지연 프록시로 주입 (임시방편) |
| 세터 주입 | 생성 시점이 아닌 이후 주입 (권장도 낮음) |

> Spring Boot 2.6+ 부터 순환 의존성은 **기본 금지**(`spring.main.allow-circular-references=false`). 발견되면 우회하지 말고 **설계를 고쳐라**.

---

## 10. AOP (관점 지향 프로그래밍) — 개념

**횡단 관심사(cross-cutting concern)** — 로깅, 트랜잭션, 보안, 성능 측정 등 여러 모듈에 흩어지는 공통 로직 — 을 **핵심 비즈니스 로직과 분리**하는 기법.

```java
@Aspect
@Component
class LoggingAspect {
    @Around("@annotation(org.springframework.web.bind.annotation.GetMapping)")
    Object logTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try { return pjp.proceed(); }                 // 실제 메서드 실행
        finally { log.info("{} took {}ms", pjp.getSignature(),
                           (System.nanoTime() - start) / 1_000_000); }
    }
}
```

- 용어: **Aspect**(관점) / **JoinPoint**(적용 지점) / **Pointcut**(대상 선정 표현식) / **Advice**(끼워 넣을 동작).
- Spring AOP는 **프록시 기반**(JDK 동적 프록시 또는 CGLIB). `@Transactional`, `@Cacheable` 도 모두 이 위에서 동작.

> [!warning] 자기 호출(self-invocation)
> 같은 빈 안에서 메서드를 직접 호출하면 프록시를 거치지 않아 AOP가 **적용되지 않는다**. `@Transactional` 이 안 먹는 흔한 원인.

---

## 관련
- [[OOP-SOLID]] — DIP/OCP를 Spring이 프레임워크 차원에서 구현
- [[Mediator-Pattern]] — 빈 간 직접 참조 제거 (순환 의존성 회피)
- [[Spring-Security]] — IoC/DI 위에 얹히는 보안 필터 체인
- [[Hexagonal-Architecture]] — DI로 포트/어댑터 경계 구성
- [[CQRS]] — Command/Query 핸들러를 빈으로 분리

## 참고
- Spring Framework Reference — Core Technologies (IoC Container)
- 토비의 스프링 — IoC/DI, AOP
