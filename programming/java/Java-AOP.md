---
tags:
  - java
  - aop
  - spring
  - proxy
  - aspect
created: 2026-06-15
---

# AOP (Aspect-Oriented Programming)

> 여러 계층에 흩어진 **횡단 관심사(Cross-Cutting Concerns)**를 모듈화하는 프로그래밍 패러다임.
> 핵심 로직(비즈니스) 코드를 건드리지 않고 로깅·트랜잭션·보안 등을 적용.

---

## 핵심 개념

| 용어 | 설명 |
|------|------|
| **Aspect** | 횡단 관심사를 모듈화한 단위 (`@Aspect` 클래스) |
| **JoinPoint** | Aspect를 적용할 수 있는 지점 (메서드 호출, 예외 등) |
| **Pointcut** | 어떤 JoinPoint에 적용할지 결정하는 표현식 |
| **Advice** | JoinPoint에서 실행할 동작 (Before, After, Around 등) |
| **Weaving** | Advice를 타겟 객체에 적용하는 과정 |

```
[Client] → [Proxy] → [Target]
              ↑
          Weaving: Aspect 코드 삽입
```

---

## Advice 종류

```java
@Aspect
@Component
public class LoggingAspect {

    // 메서드 실행 전
    @Before("execution(* com.example.service.*.*(..))")
    public void before(JoinPoint jp) {
        log.info("Before: {}", jp.getSignature());
    }

    // 메서드 정상 반환 후
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))",
                    returning = "result")
    public void afterReturning(JoinPoint jp, Object result) {
        log.info("Returned: {}", result);
    }

    // 예외 발생 후
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))",
                   throwing = "ex")
    public void afterThrowing(JoinPoint jp, Exception ex) {
        log.error("Exception in {}: {}", jp.getSignature(), ex.getMessage());
    }

    // 메서드 종료 후 (정상/예외 무관)
    @After("execution(* com.example.service.*.*(..))")
    public void after(JoinPoint jp) {
        log.info("After: {}", jp.getSignature());
    }

    // 전후 제어 가능 (가장 강력)
    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed();  // 실제 메서드 실행
            return result;
        } finally {
            log.info("{} took {}ms",
                pjp.getSignature(), System.currentTimeMillis() - start);
        }
    }
}
```

---

## Pointcut 표현식

```java
// execution(접근제어자? 반환타입 패키지?.클래스?.메서드(파라미터) throws?)
execution(* com.example.service.*.*(..))     // service 패키지 모든 메서드
execution(public * *(..))                     // 모든 public 메서드
execution(* com.example..*.*(..))            // com.example 하위 모든 패키지

// within — 타입 기준
within(com.example.service.*)               // service 패키지 내 모든 타입

// @annotation — 특정 어노테이션 붙은 메서드
@annotation(org.springframework.transaction.annotation.Transactional)

// bean — Bean 이름 기준
bean(orderService)
bean(*Service)                              // 이름이 Service로 끝나는 Bean
```

### Pointcut 재사용
```java
@Aspect
@Component
public class AppAspects {

    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Pointcut("execution(* com.example.repository.*.*(..))")
    public void repositoryLayer() {}

    @Pointcut("serviceLayer() || repositoryLayer()")
    public void appLayer() {}

    @Before("appLayer()")
    public void log(JoinPoint jp) { ... }
}
```

---

## 프록시 메커니즘

### JDK Dynamic Proxy vs CGLIB

| | JDK Dynamic Proxy | CGLIB |
|---|---|---|
| **조건** | 인터페이스 있음 | 인터페이스 없음 (클래스 직접) |
| **방식** | `java.lang.reflect.Proxy` | 바이트코드 조작으로 서브클래스 생성 |
| **final 메서드** | 해당 없음 | 적용 불가 ⚠️ |
| **Spring Boot 기본** | — | `proxyTargetClass=true` (CGLIB 우선) |

```java
// Spring Boot 기본 (CGLIB): 인터페이스 없어도 프록시 생성
@Service
public class OrderService { ... }   // 인터페이스 없어도 AOP 적용 가능

// 주의: CGLIB은 final 클래스/메서드에 AOP 적용 불가
@Service
public final class OrderService { ... }  // ❌ CGLIB 프록시 생성 실패
```

### 자기 호출(Self-Invocation) 문제
```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        // 내부에서 같은 클래스 메서드 호출 → 프록시 우회 → AOP 미적용!
        this.sendNotification(order);
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendNotification(Order order) { ... }
    // → @Transactional 동작 안 함
}
```

**해결**: 자기 자신을 주입받아 사용하거나, 별도 클래스로 분리.

---

## 실전 예시

### 1. 실행 시간 측정
```java
@Aspect @Component
@Slf4j
public class PerformanceAspect {

    @Around("@annotation(Timed)")  // 커스텀 어노테이션
    public Object measure(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        Object result = pjp.proceed();
        log.info("[{}] {}ms", pjp.getSignature().getName(),
            (System.nanoTime() - start) / 1_000_000);
        return result;
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timed {}

// 사용
@Timed
public List<Order> findAllOrders() { ... }
```

### 2. 감사 로그 (Audit)
```java
@Aspect @Component
@RequiredArgsConstructor
public class AuditAspect {
    private final AuditLogRepository auditRepo;

    @AfterReturning("@annotation(Audited)")
    public void audit(JoinPoint jp) {
        String user = SecurityContextHolder.getContext()
            .getAuthentication().getName();
        auditRepo.save(new AuditLog(user, jp.getSignature().getName(),
            LocalDateTime.now()));
    }
}
```

### 3. 예외 공통 처리
```java
@Aspect @Component
@Slf4j
public class ExceptionLoggingAspect {

    @AfterThrowing(
        pointcut = "within(com.example.service..*)",
        throwing = "ex"
    )
    public void logException(JoinPoint jp, RuntimeException ex) {
        log.error("Service exception [{}]: {}", jp.getSignature(), ex.getMessage());
        // Slack 알림, 지표 수집 등 추가 가능
    }
}
```

---

## @Transactional의 AOP 내부 동작

```
@Transactional 메서드 호출
    ↓
[TransactionInterceptor(Around Advice)]
    ↓ before
  트랜잭션 시작 (Connection.setAutoCommit(false))
    ↓ proceed
  실제 메서드 실행
    ↓ after
  정상: commit / 예외: rollback
```

- `@Transactional`은 Spring AOP(CGLIB 프록시) 로 구현
- 같은 클래스 내 자기 호출 시 트랜잭션 미적용 → 위의 Self-Invocation 문제와 동일

---

## Spring AOP vs AspectJ

| | Spring AOP | AspectJ |
|---|---|---|
| **Weaving 시점** | 런타임 (프록시) | 컴파일 타임 / 로드 타임 |
| **JoinPoint 범위** | 메서드 호출만 | 필드 접근·생성자·정적 메서드 등 |
| **self-invocation** | 미적용 ⚠️ | 적용 ✅ |
| **설정 복잡도** | 간단 (Spring 기본) | 복잡 (별도 컴파일러/에이전트) |
| **성능** | 런타임 오버헤드 | 컴파일 시 처리 → 약간 빠름 |

대부분의 Spring 애플리케이션은 **Spring AOP**로 충분.
메서드 외 JoinPoint나 self-invocation 해결이 필수면 **AspectJ** 고려.

---

## 관련
- [[Java-DI]] · [[Java-Patterns]] (Dynamic Proxy) · [[OOP-SOLID]]
