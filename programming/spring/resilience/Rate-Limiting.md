---
tags:
  - resilience
  - rate-limiting
  - api-gateway
  - redis
  - msa
created: 2026-06-17
---

# Rate Limiting (처리율 제한)

> [!summary] 한 줄 요약
> 단위 시간당 허용 요청 수를 제한해 **과부하·남용·불공정**을 막는 방어선. 알고리즘(Token Bucket 등)·적용 위치(Gateway/앱)·키 전략(per-user/IP)을 조합하며, 분산 환경에선 **Redis 공유 카운터**로 일관성을 맞춘다. 초과 시 **429 + Retry-After**.

---

## 1. 왜 필요한가
- **과부하 방지**: 트래픽 폭주가 백엔드·DB를 무너뜨리기 전에 입구에서 차단. 시스템 보호의 첫 관문.
- **공정성**: 한 사용자가 자원을 독식하지 못하게(noisy neighbor 방지). 무료/유료 티어별 쿼터.
- **비용/남용 방어**: 크롤러·brute-force·DDoS 완화, 호출당 비용이 큰 API(외부 결제, [[../../../ai/LLMOps]]) 보호.
- [[Resilience4j]]의 RateLimiter는 **나가는 호출**(외부 API 쿼터 보호)에, Gateway 제한은 **들어오는 호출**에 주로 쓴다.

## 2. 알고리즘 4가지
### 2-1. Token Bucket
일정 속도로 토큰을 채우고, 요청마다 토큰 1개 소비. 버킷 용량만큼 **버스트 허용**.

| 장점 | 단점 |
|------|------|
| 순간 버스트 허용(용량 내) | 파라미터(rate·capacity) 튜닝 필요 |
| 메모리 적음, 가장 널리 쓰임 ⭐ | — |

### 2-2. Leaky Bucket
요청을 큐에 넣고 **일정 속도로 흘려보냄**(누수). 출력률이 고정 → 평활화.

| 장점 | 단점 |
|------|------|
| 출력 트래픽 일정(다운스트림 보호) | 버스트 흡수 못 함, 큐 지연 발생 |

### 2-3. Fixed Window Counter
"1분당 100회"처럼 고정 시간창마다 카운터 리셋. 구현 단순.

| 장점 | 단점 |
|------|------|
| 가장 단순, 메모리 최소 | **경계 문제**: 창 경계에서 2배 버스트 가능 |

### 2-4. Sliding Window (Log / Counter)
- **Log**: 각 요청 타임스탬프를 저장, 최근 N초 내 개수로 판정 → 정확하지만 메모리 많이 씀.
- **Counter**: 현재 창 + 직전 창을 가중 평균 → 경계 문제 완화 + 메모리 적당(실무 균형점 ⭐).

| 알고리즘 | 버스트 | 정확도 | 메모리 | 적합 |
|----------|--------|--------|--------|------|
| Token Bucket | 허용 | 중 | 적음 | 범용 API ⭐ |
| Leaky Bucket | 불허 | 중 | 중 | 출력 평활화 필요 |
| Fixed Window | 경계서 2배 | 낮음 | 최소 | 단순·관대한 제한 |
| Sliding Log | 없음 | 높음 | 많음 | 정밀 과금 |
| Sliding Counter | 거의 없음 | 높음 | 적당 | 정밀+효율 균형 ⭐ |

## 3. 적용 위치
| 위치 | 특징 | 언제 |
|------|------|------|
| **API Gateway** | 진입점에서 일괄 차단, 백엔드 도달 전 방어 ⭐ | 전사 정책, DDoS 1차 |
| **애플리케이션** | 비즈니스 맥락(티어·엔드포인트별) 세밀 제어 | 도메인별 차등 쿼터 |
| **Redis 분산** | 여러 인스턴스가 카운터 공유 | 스케일아웃 환경(필수) |

- 단일 인스턴스 인메모리 카운터는 파드 수만큼 한도가 뻥튀기됨 → 다중 인스턴스면 **Redis 공유 카운터** 필수([[Redis]], [[../deploy/Kubernetes]]).

## 4. 구현
### 4-1. Spring Cloud Gateway + Redis
`RequestRateLimiter` 필터는 **Redis 기반 Token Bucket**을 내장.

```yaml
# application.yml ([[Spring-Cloud-Gateway]])
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10   # 초당 보충 토큰
                redis-rate-limiter.burstCapacity: 20    # 최대 버스트
                key-resolver: "#{@userKeyResolver}"     # 키 전략 빈
```
```java
@Bean
KeyResolver userKeyResolver() {                          // per-user 키
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id"));
}
```

### 4-2. Bucket4j (애플리케이션 레벨)
JVM 인메모리 또는 Redis/Hazelcast 백엔드 지원, Token Bucket 기반.

```java
Bucket bucket = Bucket.builder()
    .addLimit(limit -> limit.capacity(100)
        .refillGreedy(100, Duration.ofMinutes(1)))       // 분당 100, 그리디 보충
    .build();

if (bucket.tryConsume(1)) {
    return ResponseEntity.ok(service.handle(req));
}
return ResponseEntity.status(429)
    .header("Retry-After", "60").build();
```

### 4-3. Resilience4j RateLimiter (나가는 호출용)
```yaml
resilience4j:
  ratelimiter:
    instances:
      externalApi:
        limit-for-period: 100          # 주기당 허용 수
        limit-refresh-period: 1s
        timeout-duration: 0            # 즉시 거부(대기 안 함)
```
```java
@RateLimiter(name = "externalApi", fallbackMethod = "fallback")
public Result callExternal(Req req) { return client.call(req); }
```

## 5. 키 전략 (per-user / per-IP / global)
| 키 | 용도 | 주의 |
|----|------|------|
| **per-user** | 로그인 API, 티어별 쿼터 | 인증 정보 필요 |
| **per-IP** | 익명 트래픽, 로그인 폭주 | 프록시/NAT면 IP 공유 → `X-Forwarded-For` 신뢰 검증 |
| **per-API-key** | 외부 파트너/공개 API | 발급 단위 과금 |
| **global** | 백엔드 절대 한도 보호 | 전체 합산 상한 |

- 보통 **계층 조합**: global(시스템 보호) + per-user(공정성) + 민감 엔드포인트(로그인) 별도 강한 제한.

## 6. 분산 환경 처리 — Redis 공유 카운터
- 여러 인스턴스가 같은 키의 카운터를 Redis에서 공유해야 한도가 정확. 증가+검증을 **Lua 스크립트로 원자 처리**(read-modify-write race 방지).
- Spring Cloud Gateway의 `RequestRateLimiter`, Bucket4j-Redis 모두 내부적으로 원자 연산 사용.
- 분산 카운터 갱신은 Redis 단일 지점에 의존 → Redis 가용성·지연이 제한기 성능을 좌우. 자세한 동시성 이슈는 [[Distributed-Lock]] 참고.

## 7. 429 응답과 Retry-After
표준 거부 응답으로 클라이언트가 **언제 재시도할지** 알려준다.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60                       # 60초 후 재시도 (초 또는 HTTP-date)
X-RateLimit-Limit: 100                # 정책상 한도
X-RateLimit-Remaining: 0              # 남은 호출 수
X-RateLimit-Reset: 1718600000        # 리셋 시각(epoch)
```
- 클라이언트는 `Retry-After`를 존중해 **지수 백오프 + jitter**로 재시도. 무지성 즉시 재시도는 thundering herd 유발.

## 8. 백프레셔와의 관계
- Rate limiting은 **거부(reject)** 기반 — 한도 초과분을 버린다. **백프레셔(backpressure)**는 생산 속도 자체를 늦춰 흐름을 조절(리액티브 스트림, 메시지 컨슈머).
- 둘은 보완 관계: 입구에서 rate limit으로 1차 차단 + 내부 파이프라인에서 백프레셔로 흐름 제어. 외부 호출 폭주는 [[Resilience4j]] Bulkhead/TimeLimiter와도 함께.

## 9. LLM API rate limit 대응 사례
- OpenAI/Anthropic 등 LLM API는 **RPM(분당 요청)·TPM(분당 토큰)** 이중 한도를 둔다. 토큰 기반이라 단순 요청 수 제한만으론 부족.
- 대응: ① 요청 큐 + 토큰 버킷으로 **TPM 추정 소비** 관리, ② 429 수신 시 `Retry-After`/`x-ratelimit-reset` 헤더 기반 백오프, ③ 사용자별 쿼터를 자체 Redis 제한기로 선차단해 상위 API 한도 도달 전에 보호. 운영 패턴은 [[../../../ai/LLMOps]].

## 10. 관련
- [[Resilience4j]] · [[Spring-Cloud-Gateway]] · [[Redis]] · [[../../../ai/LLMOps]] · [[Distributed-Lock]]
