---
tags:
  - observability
  - logging
  - logback
  - slf4j
created: 2026-06-15
---

# Logging (로깅)

> [!summary] 한 줄 요약
> 애플리케이션 이벤트 기록. MSA에서는 **구조화(JSON) 로깅 + 중앙 집계 + 추적 ID 연계**가 핵심. Java는 SLF4J(API) + Logback/Log4j2(구현).

---

## 1. 기본 — SLF4J + Logback
```java
@Service
@Slf4j                              // lombok: private static final Logger log
public class OrderService {
    public void place(OrderCommand cmd) {
        log.info("주문 생성 시작 customerId={}", cmd.customerId());   // 파라미터화(문자열 결합 X)
        try {
            // ...
        } catch (Exception e) {
            log.error("주문 생성 실패 customerId={}", cmd.customerId(), e);  // 예외는 마지막 인자
        }
    }
}
```
- 레벨: `ERROR > WARN > INFO > DEBUG > TRACE`.
- ⚠️ `log.info("x=" + obj)` 금지 → `log.info("x={}", obj)` (지연 평가).

## 2. 구조화(JSON) 로깅 — 집계 친화
```xml
<!-- logback-spring.xml + logstash-logback-encoder -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```
```json
{"@timestamp":"...","level":"INFO","logger":"OrderService",
 "message":"주문 생성","traceId":"abc123","customerId":"c1"}
```
> 컨테이너(K8s)에선 **stdout으로 JSON 출력** → 수집기가 표준출력을 긁어간다(12-factor).

## 3. MDC — 컨텍스트 정보 자동 첨부
```java
MDC.put("traceId", traceId);            // 이후 모든 로그에 traceId 포함
try { ... } finally { MDC.clear(); }
```
> 추적 ID는 보통 [[Tracing]]/Micrometer가 자동으로 MDC에 넣어준다.

## 4. 중앙 집계 파이프라인
```
앱(stdout JSON) → 수집(Promtail/Fluentd/Filebeat) → 저장(Loki/Elasticsearch) → 조회(Grafana/Kibana)
```
- **Loki + Promtail + Grafana**: 가볍고 K8s 친화(라벨 기반).
- **ELK(Elasticsearch+Logstash+Kibana)**: 강력한 풀텍스트 검색.

## 5. 베스트 프랙티스
- **민감정보 마스킹**(비밀번호/토큰/주민번호 로깅 금지).
- 로그 레벨 운영 기준: 운영은 INFO, 문제 시 DEBUG 동적 변경.
- 모든 로그에 **traceId** 부여 → [[Tracing]]과 연계해 요청 단위 추적.
- 과도한 로깅은 비용/성능 부담 → 의미 있는 지점에.

## 6. 관련
- [[Tracing]] · [[Metrics]] · [[Kubernetes]] · [[MSA]]
