---
tags:
  - spring
  - gateway
  - api-gateway
  - msa
  - reactive
created: 2026-06-15
---

# Spring Cloud Gateway

> [!summary] 한 줄 요약
> Spring 진영의 **API Gateway**. 모든 클라이언트 요청의 **단일 진입점**으로, 라우팅·인증·RateLimit·CircuitBreaker·헤더조작 등 **공통 관심사**를 처리한다. WebFlux 기반 **논블로킹**.

---

## 1. 사용처 (왜 게이트웨이인가)
[[MSA]]에서 클라이언트가 수십 개 서비스를 직접 호출하면 인증·CORS·로깅·라우팅이 흩어진다. 게이트웨이가 이를 **한 곳에 집중**한다.

- 라우팅 / 로드밸런싱 (서비스 디스커버리 연동)
- 인증·인가(JWT 검증), CORS
- Rate Limiting, Circuit Breaker
- 요청/응답 변환, 로깅, 메트릭
- BFF(Backend For Frontend) 진입점

> [!note] 구조 주의
> Spring Cloud Gateway는 **WebFlux(리액티브)** 기반이다. Spring MVC와 같은 앱에 섞지 말 것. (MVC 기반이 필요하면 Gateway MVC 변형 사용)

---

## 2. 핵심 개념
- **Route**: id + 목적지 URI + Predicate + Filter 묶음.
- **Predicate**: 요청 매칭 조건 (Path, Method, Header, Host, 시간 등).
- **Filter**: 요청/응답 가공 (Pre/Post). 전역 또는 라우트별.

```
요청 → [Predicate 매칭] → [Pre Filters] → 라우팅(백엔드) → [Post Filters] → 응답
```

---

## 3. 설정 (YAML)
```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
```
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service          # lb:// → 디스커버리 로드밸런싱
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
          filters:
            - StripPrefix=1                 # /api/orders → /orders
            - name: CircuitBreaker          # Resilience4j 연동
              args:
                name: orderCB
                fallbackUri: forward:/fallback/order
            - name: RequestRateLimiter      # Redis 기반 RateLimit
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

        - id: payment-service
          uri: http://payment:8080
          predicates:
            - Path=/api/payments/**
          filters:
            - AddRequestHeader=X-Gateway, scg

      default-filters:                       # 모든 라우트 공통
        - AddResponseHeader=X-Response-Time, ${T(System).currentTimeMillis()}
```

## 4. 자바 DSL 라우팅
```java
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order", r -> r.path("/api/orders/**")
            .filters(f -> f.stripPrefix(1)
                           .circuitBreaker(c -> c.setName("orderCB")
                                                 .setFallbackUri("forward:/fallback/order")))
            .uri("lb://order-service"))
        .build();
}
```

## 5. 커스텀 Global Filter (예: JWT 인증)
```java
@Component
public class AuthFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();   // 체인 중단
        }
        // 사용자 정보를 다운스트림 헤더로 전달
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-User-Id", subjectOf(token)).build();
        return chain.filter(exchange.mutate().request(mutated).build());
    }
    @Override public int getOrder() { return -1; }   // 우선순위
}
```

## 6. Rate Limiter (Redis 기반)
```java
@Bean
KeyResolver userKeyResolver() {                  // 사용자별 제한 키
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id"));
}
```
> [[Redis]]/[[Valkey]]가 토큰버킷 상태 저장소로 쓰인다.

## 7. 베스트 프랙티스
- 게이트웨이는 **얇게** 유지(비즈니스 로직 X) — 횡단 관심사만.
- 인증은 게이트웨이에서 1차 검증, 서비스는 신뢰 헤더/토큰 재검증(zero-trust).
- 장애 전파 차단을 위해 라우트별 [[Resilience4j]] CircuitBreaker·타임아웃 적용.
- 관측성: 요청 추적 ID 부여(전파), 메트릭·로그 중앙화.

## 8. 도입 시점 — 언제 도입해야 하는가

### 도입이 필요한 신호

| 신호 | 설명 |
|---|---|
| 서비스 3개 이상 | 각 서비스마다 인증 로직 중복 구현 |
| 외부 클라이언트에 내부 서비스 직접 노출 | 내부 구조가 드러남, IP 변경 시 클라이언트 수정 필요 |
| 서비스별 CORS 설정 중복 | 각 서비스에서 따로 설정 |
| 팀별 서비스가 존재 (MSA 초기) | 단일 진입점 필요 |

### 모놀리스에서는 필요 없음

단일 Spring Boot 앱에 게이트웨이를 붙이면 불필요한 네트워크 홉만 추가. MSA로 분리되기 시작할 때 도입.

---

## 9. 장점 vs 단점

### ✅ 장점

| 항목 | 내용 |
|---|---|
| 단일 진입점 | 클라이언트는 GW 주소 하나만 알면 됨 |
| 횡단 관심사 중앙화 | 인증·Rate Limit·CORS·로깅을 한 곳에서 처리 |
| 백엔드 보호 | 내부 서비스 IP/구조 숨김 |
| 논블로킹 | WebFlux 기반 → 적은 스레드로 많은 요청 처리 |
| Spring 생태계 통합 | Resilience4j, Eureka, Zipkin 등 자연스럽게 연동 |
| 동적 라우팅 | Actuator API로 런타임 라우팅 변경 가능 |

### ❌ 단점

| 항목 | 내용 |
|---|---|
| SPOF | GW 장애 = 전체 서비스 장애 → 반드시 다중화 |
| WebFlux 필수 | 리액티브 프로그래밍 학습 필요. MVC 코드와 섞기 어려움 |
| 디버깅 어려움 | 비동기 스택 트레이스, 컨텍스트 전파 복잡 |
| 성능 오버헤드 | 모든 요청이 GW를 경유 → 추가 지연 (~1~5ms) |
| 비즈니스 로직 오염 | GW에 로직이 쌓이면 관리 지옥 → 얇게 유지 필수 |

---

## 10. Spring Cloud Gateway vs 대안 비교

| | Spring Cloud Gateway | Netflix Zuul 1 | Nginx | Kong |
|---|---|---|---|---|
| 방식 | 논블로킹 (WebFlux) | 블로킹 (Servlet) | 이벤트 기반 (C) | Nginx + Lua |
| 성능 | 높음 | 낮음 | **최고** | 높음 |
| Java 통합 | ✅ 완벽 | ✅ 완벽 | 제한적 | 제한적 |
| 커스텀 로직 | Java/Kotlin | Java/Kotlin | Nginx 모듈 | Lua 플러그인 |
| 관리 UI | ❌ (Actuator) | ❌ | 없음 | ✅ Admin API |
| k8s 통합 | 보통 | 보통 | ✅ Ingress Controller | ✅ Ingress |
| 권장 상황 | **Spring MSA** | 레거시만 | 고성능 순수 프록시 | 언어 무관 다중 팀 |

> [!tip] 언제 Nginx/Kong을 선택하는가
> - 팀이 Java 외 다른 언어를 쓴다면 Kong이 더 중립적
> - 클러스터에 이미 Nginx Ingress Controller가 있다면 SCG 없이 Nginx에 플러그인
> - SCG는 Spring Boot 팀이 Java로 게이트웨이 로직을 완전히 제어하고 싶을 때 최적

---

## 11. 관련
- [[Resilience4j]] · [[MSA]] · [[Redis]] · [[Valkey]]
- 인프라 관점 API GW: [[API-Gateway]]
