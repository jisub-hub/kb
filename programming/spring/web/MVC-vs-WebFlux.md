---
tags:
  - spring
  - mvc
  - webflux
  - reactive
  - concurrency
created: 2026-06-15
---

# Spring MVC vs Spring WebFlux

> [!summary] 한 줄 요약
> **MVC**: 전통적 **블로킹(Thread-per-Request)** 웹 스택, Servlet 기반. **WebFlux**: **논블로킹/리액티브** 스택, Reactor(Mono/Flux) + 이벤트 루프 기반. "WebFlux가 항상 빠르다"는 오해 — 워크로드에 따라 고른다.

---

## 1. 근본 차이 — 스레드 모델

### Spring MVC (Servlet, Blocking)
```
요청1 → Thread1 ─ (DB/외부 I/O 대기 동안 스레드 점유) ─► 응답
요청2 → Thread2 ─ ...
   동시성 = 스레드 풀 크기에 제한 (기본 Tomcat 200)
```

### Spring WebFlux (Reactive, Non-blocking)
```
요청 N개 → 소수의 이벤트 루프 스레드(보통 CPU 코어 수)
          I/O 대기 중엔 스레드를 놓고 다른 요청 처리
   적은 스레드로 매우 높은 동시성 (단, 전 구간 논블로킹이어야 함)
```

---

## 2. 비교표

| 항목 | Spring MVC | Spring WebFlux |
|------|-----------|----------------|
| 기반 | Servlet API | Reactive Streams (Reactor) |
| 런타임 | Tomcat/Jetty/Undertow(서블릿) | Netty(기본)/서블릿3.1+ |
| I/O | 블로킹 | 논블로킹 |
| 스레드 모델 | thread-per-request | event loop |
| 반환 타입 | 객체, `ResponseEntity` | `Mono<T>`, `Flux<T>` |
| 동시성(I/O 바운드) | 스레드 풀 한계 | 매우 높음 |
| DB 접근 | JDBC/JPA(블로킹) | [[R2DBC]], reactive mongo 등 |
| 프로그래밍 난도 | 쉬움 | 높음(리액티브 사고) |
| 디버깅/스택트레이스 | 직관적 | 어려움 |
| 생태계 성숙도 | 매우 높음 | 발전 중 |
| 블로킹 코드 혼입 | 문제 없음 | **치명적**(이벤트 루프 마비) |

---

## 3. 코드 비교

### 컨트롤러
```java
// MVC — 값을 직접 반환 (블로킹)
@RestController
@RequiredArgsConstructor
class MvcController {
    private final UserService service;
    @GetMapping("/users/{id}")
    public User get(@PathVariable Long id) {
        return service.findById(id);          // 스레드가 결과까지 대기
    }
}

// WebFlux — Mono/Flux 반환 (논블로킹)
@RestController
@RequiredArgsConstructor
class FluxController {
    private final UserReactiveService service;
    @GetMapping("/users/{id}")
    public Mono<User> get(@PathVariable Long id) {
        return service.findById(id);          // 구독 시점에 비동기 실행
    }
    @GetMapping(value = "/users", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<User> stream() {
        return service.streamAll();           // 서버센트이벤트 스트리밍
    }
}
```

### WebFlux 함수형 라우팅 (대안 스타일)
```java
@Bean
RouterFunction<ServerResponse> routes(UserHandler handler) {
    return route(GET("/users/{id}"), handler::get)
          .andRoute(POST("/users"), handler::create);
}
```

### ⚠️ WebFlux에서 절대 하면 안 되는 것
```java
public Mono<User> bad(Long id) {
    User u = jpaRepository.findById(id).get();   // ❌ 블로킹 JDBC → 이벤트 루프 마비
    return Mono.just(u);
}
// 불가피하게 블로킹 호출이 필요하면 별도 스케줄러로 격리:
return Mono.fromCallable(() -> jpaRepository.findById(id).orElseThrow())
           .subscribeOn(Schedulers.boundedElastic());
```

---

## 4. 언제 무엇을 쓰나

> [!success] Spring MVC — 대부분의 경우
> - 일반 CRUD/비즈니스 웹앱, REST API
> - 팀 생산성·유지보수·풍부한 생태계가 중요
> - JPA/MyBatis 등 블로킹 DB 사용
> - **Java 21 가상 스레드**와 결합 시, 블로킹 코드 그대로 높은 동시성 확보 가능 → WebFlux 필요성 감소

> [!success] Spring WebFlux — 특정 조건
> - **매우 높은 동시 연결 + I/O 바운드**(게이트웨이, 프록시, 채팅, 스트리밍)
> - 엔드투엔드 논블로킹 가능(R2DBC, reactive 클라이언트)
> - 백프레셔가 필요한 스트림 처리
> - [[Spring-Cloud-Gateway]] 는 WebFlux 기반

> [!warning] 흔한 오해
> - "리액티브 = 항상 고성능" ❌ — CPU 바운드/평범한 트래픽엔 이점 없고 복잡도만 증가.
> - 한 군데라도 블로킹(JDBC, `.block()`)이 섞이면 오히려 **MVC보다 느려진다**.
> - 관련: [[Sync-vs-Async-DataAccess]]

---

## 5. 가상 스레드(Virtual Threads, Java 21+)
- MVC + 가상 스레드(Spring Boot 3.2, `spring.threads.virtual.enabled=true`)로 **블로킹 코드를 유지하면서** 높은 동시성.
- "리액티브의 복잡함 없이 동시성"을 원하면 다수 케이스에서 WebFlux의 현실적 대안.
- 📋 둘의 성능 차이를 깊게 비교한 리포트: [[VirtualThreads-vs-WebFlux]]

## 6. 관련
- [[Sync-vs-Async-DataAccess]] · [[R2DBC]] · [[Spring-Cloud-Gateway]] · [[REST-API]] · [[gRPC]]
