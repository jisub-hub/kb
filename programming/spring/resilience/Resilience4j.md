---
tags:
  - spring
  - resilience4j
  - resilience
  - msa
created: 2026-06-15
---

# Resilience4j

> [!summary] 한 줄 요약
> 경량 **회복탄력성(fault tolerance) 라이브러리**. 분산 시스템에서 장애가 전파되지 않도록 **Circuit Breaker / Retry / Rate Limiter / Bulkhead / TimeLimiter** 를 애너테이션·함수형으로 제공. (Netflix Hystrix의 사실상 후계자)

---

## 1. 왜 필요한가 (사용처)
[[MSA]]에서 서비스 A가 느린/죽은 서비스 B를 계속 호출하면 → A의 스레드가 묶이고 → **장애가 연쇄 전파(cascading failure)**. Resilience4j는 이를 차단한다.

- 외부 API / 다른 마이크로서비스 호출
- DB·캐시 등 외부 자원 접근
- 불안정한 네트워크 경로

## 2. 5가지 핵심 모듈

| 모듈 | 역할 | 언제 |
|------|------|------|
| **CircuitBreaker** | 실패율 임계 초과 시 회로를 열어 호출 차단(빠른 실패) | 장애 서비스 격리 |
| **Retry** | 일시적 실패 재시도 | 순간적 네트워크 오류 |
| **RateLimiter** | 단위 시간당 호출 수 제한 | 외부 API 쿼터 보호 |
| **Bulkhead** | 동시 호출 수 격리 | 한 자원이 전체 스레드 점유 방지 |
| **TimeLimiter** | 호출 타임아웃 | 무한 대기 방지 |

### Circuit Breaker 상태
```
CLOSED ──(실패율 초과)──► OPEN ──(대기시간 경과)──► HALF_OPEN
   ▲                                                    │
   └──────────(시험 호출 성공)──────────────────────────┘
            (시험 호출 실패 시 다시 OPEN)
```

---

## 3. Spring Boot 설정
```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
implementation 'org.springframework.boot:spring-boot-starter-aop'      // 애너테이션 사용 시
implementation 'org.springframework.boot:spring-boot-starter-actuator' // 상태 모니터링
```
```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        failure-rate-threshold: 50          # 실패율 50% 초과 → OPEN
        wait-duration-in-open-state: 10s    # OPEN 유지 시간
        permitted-number-of-calls-in-half-open-state: 3
        slow-call-duration-threshold: 2s
        slow-call-rate-threshold: 80
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
  ratelimiter:
    instances:
      paymentService:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 0
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 20
  timelimiter:
    instances:
      paymentService:
        timeout-duration: 3s
```

## 4. 애너테이션 사용법
```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentClient paymentClient;       // Feign 등

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
    @Retry(name = "paymentService")                  // 적용 순서: Bulkhead → TimeLimiter → RateLimiter → CircuitBreaker → Retry
    @Bulkhead(name = "paymentService")
    public PaymentResult charge(ChargeRequest req) {
        return paymentClient.charge(req);
    }

    // fallback: 마지막 파라미터로 예외를 받음 (시그니처 일치 필요)
    private PaymentResult fallback(ChargeRequest req, Throwable t) {
        log.warn("결제 서비스 장애, 폴백 실행: {}", t.getMessage());
        return PaymentResult.pending();              // 대체 응답/큐 적재
    }
}
```

### TimeLimiter는 비동기와 함께
```java
@TimeLimiter(name = "paymentService")
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackAsync")
public CompletableFuture<PaymentResult> chargeAsync(ChargeRequest req) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
}
```

## 5. 함수형(프로그래매틱) 사용법
```java
CircuitBreaker cb = CircuitBreaker.ofDefaults("payment");
Supplier<PaymentResult> decorated =
    CircuitBreaker.decorateSupplier(cb, () -> paymentClient.charge(req));
PaymentResult result = Try.ofSupplier(decorated)
                          .recover(t -> PaymentResult.pending())
                          .get();
```

## 6. Prometheus + Grafana 모니터링

```yaml
# application.yml — Actuator + Micrometer 노출
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true
```

```yaml
# prometheus.yml — Spring 앱 스크레이프 설정
scrape_configs:
  - job_name: 'spring-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['myapp:8080']
    scrape_interval: 15s
```

### Actuator 엔드포인트 확인

```bash
# Circuit Breaker 현재 상태
curl http://localhost:8080/actuator/circuitbreakers
# {
#   "circuitBreakers": {
#     "paymentService": {
#       "failureRate": "12.5%",
#       "state": "CLOSED",
#       "bufferedCalls": 20
#     }
#   }
# }

# 최근 이벤트 (OPEN/CLOSE/HALF_OPEN 전환 이력)
curl http://localhost:8080/actuator/circuitbreakerevents/paymentService
```

### Resilience4j 핵심 Prometheus 메트릭

```
# Circuit Breaker 상태 (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
resilience4j_circuitbreaker_state{name="paymentService"}

# 실패율
resilience4j_circuitbreaker_failure_rate{name="paymentService"}

# 느린 호출 비율
resilience4j_circuitbreaker_slow_call_rate{name="paymentService"}

# 총 호출 수 (성공/실패/무시됨/차단됨)
resilience4j_circuitbreaker_calls_total{name="paymentService",kind="successful"}
resilience4j_circuitbreaker_calls_total{name="paymentService",kind="failed"}
resilience4j_circuitbreaker_calls_total{name="paymentService",kind="not_permitted"}

# Retry 시도 횟수
resilience4j_retry_calls_total{name="paymentService",kind="successful_without_retry"}
resilience4j_retry_calls_total{name="paymentService",kind="successful_with_retry"}
resilience4j_retry_calls_total{name="paymentService",kind="failed_with_retry"}
```

### Grafana PromQL — 대시보드 패널

```promql
# 패널 1: Circuit Breaker 상태 (State Timeline)
resilience4j_circuitbreaker_state{application="myapp"}

# 패널 2: 실패율 추이 (Time Series)
resilience4j_circuitbreaker_failure_rate{application="myapp",name="paymentService"}

# 패널 3: 초당 차단된 호출 (Call Blocked Rate)
rate(resilience4j_circuitbreaker_calls_total{kind="not_permitted"}[1m])

# 패널 4: Retry로 인한 추가 부하 비율
rate(resilience4j_retry_calls_total{kind="successful_with_retry"}[5m])
/ rate(resilience4j_retry_calls_total[5m])

# 알람: Circuit Breaker OPEN 상태 1분 이상 지속
resilience4j_circuitbreaker_state{name="paymentService"} == 1
```

### Grafana 알람 설정 예시

```yaml
# alerting rule (Grafana Alerting 또는 Prometheus Alert)
groups:
  - name: resilience4j
    rules:
      - alert: CircuitBreakerOpen
        expr: resilience4j_circuitbreaker_state == 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Circuit Breaker OPEN: {{ $labels.name }}"
          description: "{{ $labels.application }}의 {{ $labels.name }} CB가 1분 이상 OPEN 상태"

      - alert: HighFailureRate
        expr: resilience4j_circuitbreaker_failure_rate > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "실패율 높음: {{ $labels.name }} = {{ $value }}%"
```

## 7. 베스트 프랙티스
- **Fallback은 의미 있게**: 캐시된 값, 기본값, 큐 적재 등. 단순 null 반환 지양.
- **Retry + CircuitBreaker 조합 주의**: 재시도가 장애를 증폭시키지 않도록 횟수·백오프 신중히.
- 타임아웃 없는 외부 호출 금지 → TimeLimiter 필수.
- [[Spring-Cloud-Gateway]] 라우트 레벨에서도 CircuitBreaker 필터 적용 가능.

## 8. 분산 환경 서킷브레이커 조정

앞 절의 CircuitBreaker는 **단일 인스턴스(프로세스) 안의 로컬 상태**다. 인스턴스가 8대 떠 있는 운영 환경에서는 로컬 CB만으로는 해결되지 않는 문제가 생긴다.

### 8.1 로컬 CB의 한계 — 전역 뷰가 없다
Resilience4j의 CB 상태(CLOSED/OPEN/HALF_OPEN)는 **JVM 인스턴스별로 독립**이다.
- 인스턴스 A에서 다운스트림 실패율이 임계를 넘어 CB가 **OPEN**으로 가도, **로드밸런서는 나머지 7대로 계속 트래픽을 보낸다**.
- 7대는 각자 다시 실패를 누적하고 OPEN으로 가기까지 **시차**가 있다 → 클러스터 전체로 보면 보호가 들쭉날쭉.
- 어디에도 **"전역 CB 상태"** 가 없다. `/actuator/circuitbreakers`는 그 인스턴스의 뷰일 뿐.

> 즉 로컬 CB는 "**이 인스턴스가 죽어가는 다운스트림에 매달리지 않게**" 하는 자기 보호 장치이지, 클러스터 차원의 차단기가 아니다.

### 8.2 L7 로드밸런서 헬스체크와의 상호작용
ALB/L7 LB의 헬스체크와 CB는 **서로를 모른다.**
- **타이밍 불일치**: ALB 헬스체크 주기(예: 30s, unhealthy threshold 등)와 CB의 `wait-duration-in-open-state`(예: 10s)·half-open 복구 타이밍이 어긋난다. CB는 이미 half-open으로 시험 호출을 하는데 LB는 아직 인스턴스를 정상으로 본다(혹은 그 반대).
- **헬스체크가 CB 상태를 모른다**: `/health`가 200을 주면 CB가 다운스트림 때문에 OPEN이어도 LB는 트래픽을 보낸다. 반대로, CB OPEN을 `/health`에 반영하면 **"다운스트림 한 곳이 죽었다고 인스턴스 전체를 LB 풀에서 빼는"** 과잉 격리가 일어날 수 있다 → 보통은 분리하는 게 안전.

### 8.3 썬더링 허드 (Thundering Herd)
다운스트림이 회복되는 순간이 가장 위험하다.
- 모든 인스턴스의 CB가 비슷한 시점에 OPEN으로 갔다면, `wait-duration` 경과 후 **8대가 거의 동시에 half-open → close** 로 전환된다.
- 그 순간 **억눌렸던 트래픽이 한꺼번에 다운스트림으로 폭주** → 갓 회복한 서비스가 다시 무너진다(재장애 루프).

완화:
- **Jitter**: `wait-duration`·재시도 백오프에 무작위 지터를 더해 복구 시점을 분산.
- **점진 복구**: half-open에서 한 번에 푸는 호출 수(`permitted-number-of-calls-in-half-open-state`)를 작게, 그리고 정상화되면 단계적으로 트래픽을 늘리는 램프업.
- 다운스트림 앞단의 [[Rate-Limiting]]으로 회복 직후 유입을 상한.

### 8.4 전역 상태 공유 — 할 것인가
| 방식 | 장점 | 비용 |
|------|------|------|
| **인스턴스 독립(기본)** | 단순, 외부 의존 없음, 빠름 | 전역 뷰 없음(eventual) |
| **공유 상태(Redis 등)** | 한 인스턴스의 OPEN을 전체가 즉시 공유 | 강한 일관성 비용·지연, Redis가 새 SPOF, CB가 외부 호출에 의존하는 역설 |

- 대부분은 **인스턴스 독립이 현실적**이다. 공유 CB 상태는 "강한 일관성"을 얻으려다 CB 자체가 외부 의존(느려지거나 죽으면?)을 갖는 역설에 빠진다.
- 대신 **메트릭은 중앙 집계**한다: 각 인스턴스의 `resilience4j_circuitbreaker_state`를 Prometheus로 모아 **클러스터 전역 뷰**를 *관찰*하고 알람한다(상태를 공유하진 않되 *가시성*은 확보). → [[SLO-SLI]].

### 8.5 대응 원칙
- **다운스트림 보호는 단일 장치가 아니라 조합으로**: CB(빠른 실패) + Bulkhead(자원 격리) + Rate Limit(유입 상한). 한 가지로는 썬더링 허드·자원 고갈을 다 못 막는다.
- **LB 헬스체크는 CB와 별개 신호로 유지**: 헬스체크는 "이 인스턴스 자체가 살아있나(liveness)"를, CB는 "다운스트림이 건강한가"를 본다. 둘을 섞으면 다운스트림 장애가 인스턴스 격리로 잘못 번역된다.
- **복구는 점진적으로**: jitter + half-open 제한 + 램프업으로 동시 복구 폭주를 흡수.
- 게이트웨이 레벨([[Spring-Cloud-Gateway]])·서비스 메시에서도 같은 원칙이 적용된다 → [[../architecture/MSA]] 전반의 회복탄력성 전략과 함께 설계.

### 관련 개념 링크
[[Rate-Limiting]] · [[MVC-vs-WebFlux]] · [[../architecture/MSA]] · [[SLO-SLI]]

---

## 9. 관련
- [[MSA]] · [[Spring-Cloud-Gateway]] · [[Saga-Pattern]]
- [[HTTP-Client-Resilience]] — HTTP 클라이언트 타임아웃·풀과 서킷/벌크헤드 결합
