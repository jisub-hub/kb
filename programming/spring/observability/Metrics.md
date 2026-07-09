---
tags:
  - observability
  - metrics
  - prometheus
  - grafana
  - micrometer
created: 2026-06-15
---

# Metrics (메트릭 / 모니터링)

> [!summary] 한 줄 요약
> 시스템 상태를 **수치 시계열**로 측정(요청 수, 지연, 에러율, CPU/메모리 등). Java는 **Micrometer**(파사드) → Prometheus 등으로 노출, Grafana로 시각화·알림.

---

## 1. 메트릭 종류
| 타입 | 의미 | 예 |
|------|------|-----|
| **Counter** | 증가만 | 요청 총수, 에러 수 |
| **Gauge** | 오르내림 | 큐 길이, 활성 커넥션 |
| **Timer** | 횟수+소요시간 | API 응답시간 |
| **Distribution Summary** | 값 분포 | 페이로드 크기 |

## 2. Spring — Micrometer + Prometheus
```groovy
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    tags:
      application: order-service     # 모든 메트릭 공통 태그
```
→ `/actuator/prometheus` 엔드포인트로 노출, Prometheus가 scrape.

## 3. 커스텀 메트릭
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final MeterRegistry registry;

    public void place(OrderCommand cmd) {
        registry.counter("orders.placed", "channel", cmd.channel()).increment();  // Counter

        Timer.Sample sample = Timer.start(registry);                                // Timer
        try {
            doPlace(cmd);
        } finally {
            sample.stop(registry.timer("orders.place.latency"));
        }
    }
}

// 선언적 타이머
@Timed(value = "orders.get", percentiles = {0.5, 0.95, 0.99})
public Order get(Long id) { ... }
```

## 4. 스택 & 대시보드
```
앱(/actuator/prometheus) → Prometheus(수집/저장) → Grafana(대시보드/알림)
                                                → Alertmanager(알림 라우팅)
```
- **RED 메서드**(서비스): Rate, Errors, Duration.
- **USE 메서드**(리소스): Utilization, Saturation, Errors.

## 5. 알림(Alerting) 예시 (PromQL)
```promql
# 5분간 5xx 에러율 5% 초과
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
```

## 6. 프로덕션 함정 심화 ⭐

> [!danger] Prometheus 운영 장애 1순위 = **카디널리티 폭발**
> 메트릭 설계 실수는 대시보드가 아니라 **Prometheus 서버 자체를 죽인다**(OOM·쿼리 타임아웃).

### 6.1 카디널리티 폭발 (Cardinality Explosion)

```
시계열 개수 = 메트릭 이름 × (레이블1 값 수 × 레이블2 값 수 × ...)

http_requests_total{path, method, status, userId}
   path 50 × method 5 × status 10 × userId 100,000
   = 2.5억 시계열 → Prometheus 메모리 폭발·쿼리 폭사 💥
```
- **고카디널리티 레이블 금지**: `userId`·`email`·`requestId`·`전체 URL path`·타임스탬프 — 값이 무한히 늘어나는 것.
- path는 **템플릿화**: `/orders/12345` → `/orders/{id}` (Spring `http.server.requests`는 `uri` 태그로 자동 템플릿화 — 직접 raw path 넣지 말 것).
- 진단:
```promql
# 메트릭별 시계열 수 Top — 어떤 메트릭이 폭발하는지
topk(10, count by (__name__)({__name__=~".+"}))
count(http_server_requests_seconds_count) by (job)   # 특정 메트릭 시계열 수
```
> [!tip] 레이블 추가 전 자문: "이 레이블의 값 종류가 **유한하고 작은가**(수십~수백)?" 아니면 그건 메트릭이 아니라 **로그/추적**에 담아야 한다.

### 6.2 Histogram vs Summary — 분위수 함정

```
Summary(클라이언트 분위수 계산):
  앱이 p95/p99를 미리 계산해 노출 → ⚠️ 여러 인스턴스 값을 "합산 불가"
  (p99들의 평균은 전체 p99가 아님) → 다중 인스턴스 환경에서 의미 없음

Histogram(버킷 카운트 노출):
  버킷별 누적 카운트만 노출 → 서버(PromQL)에서 합산 후 분위수 추정 가능
  histogram_quantile(0.99, sum(rate(..._bucket[5m])) by (le))
  ⚠️ 버킷 경계(le) 설정이 정확도 좌우 — SLO 임계값 근처에 버킷 배치
```
- **다중 인스턴스(K8s 다중 pod) → 거의 항상 Histogram.** Summary는 단일 인스턴스·합산 불필요할 때만.
- **Native Histogram**(Prometheus 2.40+): 버킷을 동적·고해상도로 — 버킷 수동 설정 부담↓, 저장 효율↑(점진 도입 중).

### 6.3 rate() 윈도우와 scrape interval

```
rate(metric[window]) 의 window는 scrape interval의 최소 4배 이상
  scrape 15s인데 rate(...[15s]) → 샘플 1~2개 → NaN/들쭉날쭉
  권장: rate(...[1m]) 이상 (스파이크엔 짧게, 안정엔 길게 트레이드오프)
```
- 알림용은 너무 짧으면 노이즈, 너무 길면 둔감 → SLO burn-rate에 맞춰 멀티윈도우([[SLO-SLI]]).

### 6.4 Counter reset·increase() 함정

```
앱 재시작 → counter가 0으로 리셋
  rate()/increase()는 reset을 자동 감지·보정(단조 증가 가정)
  ⚠️ raw counter 값을 그대로 그래프에 쓰면 재시작 시 절벽 → 항상 rate/increase 경유
```

### 6.5 비용 절감 — Recording Rule·다운샘플링

```
무거운 쿼리(대시보드가 매번 수억 시계열 집계) → Recording Rule로 사전 계산
  record: job:http_requests:rate5m  →  대시보드는 이 결과만 조회(빠름·저렴)
장기 보존: 다운샘플링(Thanos/Mimir/VictoriaMetrics) — 오래된 데이터 해상도↓
```

## 7. 베스트 프랙티스
- 카디널리티 폭발 주의(태그에 userId 같은 고유값 넣지 말 것 → §6.1).
- SLI/SLO 정의 후 핵심 지표 위주로([[SLO-SLI]]).
- 메트릭(수치)·로그(상세)·추적(흐름) **상관 분석** → 3축 연계([[Logging]], [[Tracing]]).
- [[Resilience4j]]·[[Kafka]] lag 등도 메트릭으로 노출.

## 8. 관련
- [[Logging]] · [[Tracing]] · [[SLO-SLI]] · [[Kubernetes]] · [[MSA]] · [[Resilience4j]]
- **인프라 구성**: [[Prometheus-Grafana]] — Helm 설치, scrape_config, Alertmanager 룰, 추천 대시보드 ID
