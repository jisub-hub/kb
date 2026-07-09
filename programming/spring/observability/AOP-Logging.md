---
tags:
  - spring
  - aop
  - logging
  - audit
  - mdc
  - slow-query
  - observability
created: 2026-06-16
---

# AOP 기반 감사 로깅 & Slow Query 탐지

> [!summary] 한 줄 요약
> AOP로 `@Controller`·`@RestController` 진입점을 가로채 **MDC traceId + JWT Actor + 요청 파라미터/Body**를 로그에 기록한다. JPA Slow Query는 1초 이상 쿼리를 **Micrometer Observation + 별도 로거**로 분리해 모니터링 수혜를 받는다.

---

## 1. 감사 로깅 구조

```
HTTP 요청 진입
  │
  ├── Filter: traceId를 MDC에 주입 (요청 전 범위)
  │
  ├── AOP (Around): Controller 메서드 진입 전/후
  │     ├── JWT에서 Actor(userId, email) 추출
  │     ├── 메서드명, 파라미터, RequestBody 수집
  │     ├── 실행 시간 측정
  │     └── 감사 로그 기록 (구조화 JSON)
  │
  └── Filter: MDC clear (요청 후)

[MDC 키]
  traceId  : 요청 단위 추적 ID
  userId   : JWT에서 추출한 사용자 ID
  email    : JWT에서 추출한 이메일
```

---

## 2. TraceId MDC 필터

```java
@Component
@Order(1)
public class TraceIdFilter extends OncePerRequestFilter {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        String traceId = Optional.ofNullable(request.getHeader(TRACE_ID_HEADER))
            .filter(StringUtils::hasText)
            .orElse(UUID.randomUUID().toString().replace("-", "").substring(0, 16));

        MDC.put("traceId", traceId);
        response.setHeader(TRACE_ID_HEADER, traceId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();    // 스레드 풀 환경에서 반드시 제거
        }
    }
}
```

---

## 3. AOP 감사 로깅 Aspect

```java
@Aspect
@Component
@Slf4j
@RequiredArgsConstructor
public class AuditLoggingAspect {

    private final ObjectMapper objectMapper;

    // @RestController + @Controller 모두 포인트컷
    @Pointcut("within(@org.springframework.web.bind.annotation.RestController *)" +
              " || within(@org.springframework.stereotype.Controller *)")
    public void controllerMethods() {}

    @Around("controllerMethods()")
    public Object logRequest(ProceedingJoinPoint pjp) throws Throwable {
        long startMs = System.currentTimeMillis();
        String methodName = pjp.getSignature().toShortString();

        // JWT Actor 추출 (Spring Security Context)
        extractAndSetActor();

        // 파라미터 수집
        String params = extractParams(pjp);

        Object result = null;
        Throwable thrown = null;
        try {
            result = pjp.proceed();
            return result;
        } catch (Throwable t) {
            thrown = t;
            throw t;
        } finally {
            long elapsed = System.currentTimeMillis() - startMs;

            if (thrown != null) {
                log.warn("[AUDIT] method={} params={} elapsed={}ms error={}",
                    methodName, params, elapsed, thrown.getMessage());
            } else {
                log.info("[AUDIT] method={} params={} elapsed={}ms",
                    methodName, params, elapsed);
            }

            // Actor MDC 제거 (traceId는 Filter가 관리)
            MDC.remove("userId");
            MDC.remove("email");
        }
    }

    private void extractAndSetActor() {
        try {
            var auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth != null && auth.getPrincipal() instanceof UserPrincipal principal) {
                MDC.put("userId", String.valueOf(principal.getUserId()));
                MDC.put("email", principal.getEmail());
            }
        } catch (Exception ignored) {
            // 인증 정보 없는 경우 (공개 API)
        }
    }

    private String extractParams(ProceedingJoinPoint pjp) {
        MethodSignature sig = (MethodSignature) pjp.getSignature();
        String[] paramNames = sig.getParameterNames();
        Object[] args = pjp.getArgs();

        Map<String, Object> paramMap = new LinkedHashMap<>();
        for (int i = 0; i < paramNames.length; i++) {
            Object arg = args[i];
            // HttpServletRequest/Response 제외, @RequestBody 직렬화
            if (arg instanceof HttpServletRequest || arg instanceof HttpServletResponse) {
                continue;
            }
            if (arg instanceof MultipartFile file) {
                paramMap.put(paramNames[i], "MultipartFile[" + file.getOriginalFilename() + "]");
            } else {
                paramMap.put(paramNames[i], sanitize(arg));
            }
        }

        try {
            return objectMapper.writeValueAsString(paramMap);
        } catch (Exception e) {
            return paramMap.toString();
        }
    }

    private Object sanitize(Object value) {
        if (value == null) return null;
        String str = value.toString();
        // 민감 필드 마스킹 (password, token, secret 등)
        if (str.toLowerCase().contains("password") ||
            str.toLowerCase().contains("token") ||
            str.toLowerCase().contains("secret")) {
            return "***";
        }
        // 긴 문자열 트리밍
        if (str.length() > 500) {
            return str.substring(0, 500) + "...(truncated)";
        }
        return value;
    }
}
```

---

## 4. 감사 로그 출력 예시 (JSON)

```json
{
  "@timestamp": "2026-06-16T10:23:45.123Z",
  "level": "INFO",
  "logger": "AuditLoggingAspect",
  "traceId": "a1b2c3d4e5f6g7h8",
  "userId": "1042",
  "email": "hong@example.com",
  "message": "[AUDIT] method=OrderController.createOrder(..) params={\"orderRequest\":{\"productId\":101,\"qty\":2}} elapsed=45ms"
}
```

---

## 5. @RequestBody를 AOP에서 읽는 문제

`HttpServletRequest.getInputStream()`은 한 번만 읽을 수 있다. AOP에서 Body를 읽으면 컨트롤러에서 못 읽는다.

```java
// ContentCachingRequestWrapper로 해결
@Bean
public FilterRegistrationBean<ContentCachingFilter> contentCachingFilter() {
    return new FilterRegistrationBean<>(new ContentCachingFilter());
}

// 또는 직접 구현
@Component
@Order(2)
public class RequestCachingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {
        // Body를 캐싱하는 Wrapper로 교체
        ContentCachingRequestWrapper wrappedRequest =
            new ContentCachingRequestWrapper(request);
        chain.doFilter(wrappedRequest, response);
    }
}

// AOP에서 Body 읽기
private String extractBody(ProceedingJoinPoint pjp) {
    // @RequestBody 파라미터를 직접 직렬화 (더 안전)
    for (Object arg : pjp.getArgs()) {
        if (arg != null && isRequestBody(arg)) {
            try {
                return objectMapper.writeValueAsString(arg);
            } catch (Exception e) {
                return arg.toString();
            }
        }
    }
    return "";
}
```

---

## 6. Slow Query 탐지

### JPA + Hibernate 쿼리 시간 측정

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        session_factory:
          statement_inspector: com.example.SlowQueryInspector

logging:
  level:
    slow-query: WARN    # 별도 로거로 관리
```

```java
// Hibernate StatementInspector로 쿼리 + 시간 측정
@Component
public class SlowQueryInspector implements StatementInspector {

    private static final Logger slowQueryLog =
        LoggerFactory.getLogger("slow-query");
    private static final long SLOW_THRESHOLD_MS = 1000;

    // ThreadLocal로 실행 시작 시간 전달
    private static final ThreadLocal<Long> queryStart = new ThreadLocal<>();

    @Override
    public String inspect(String sql) {
        queryStart.set(System.currentTimeMillis());
        return sql;
    }
}

// Hibernate EventListener로 실행 후 시간 측정
@Component
public class SlowQueryEventListener
        implements PostLoadEventListener, PreLoadEventListener {

    private static final Logger slowQueryLog =
        LoggerFactory.getLogger("slow-query");
    private static final long THRESHOLD_MS = 1000;

    @Override
    public void onPreLoad(PreLoadEvent event) {
        MDC.put("queryStart", String.valueOf(System.currentTimeMillis()));
    }

    @Override
    public void onPostLoad(PostLoadEvent event) {
        String startStr = MDC.get("queryStart");
        if (startStr != null) {
            long elapsed = System.currentTimeMillis() - Long.parseLong(startStr);
            if (elapsed >= THRESHOLD_MS) {
                slowQueryLog.warn("[SLOW_QUERY] entity={} elapsed={}ms traceId={}",
                    event.getEntity().getClass().getSimpleName(),
                    elapsed,
                    MDC.get("traceId"));
            }
            MDC.remove("queryStart");
        }
    }
}
```

### P6Spy로 실제 SQL + 바인딩 파라미터 측정 (더 정확)

```xml
<!-- pom.xml -->
<dependency>
  <groupId>com.github.gavlyukovskiy</groupId>
  <artifactId>p6spy-spring-boot-starter</artifactId>
  <version>1.9.1</version>
</dependency>
```

```yaml
# application.yml
decorator:
  datasource:
    p6spy:
      enable-logging: true
      multiline: false
      logging: slf4j
      log-format: "%(executionTime) ms | %(sql)"
```

```java
// 커스텀 P6Spy 리스너 — 1초 이상만 별도 로거
@Component
public class SlowQueryP6SpyListener implements JdbcEventListener {

    private static final Logger slowLog = LoggerFactory.getLogger("slow-query");
    private static final long THRESHOLD_MS = 1000;

    @Override
    public void onAfterAnyExecute(StatementInformation si, long elapsed, SQLException e) {
        if (elapsed >= THRESHOLD_MS) {
            slowLog.warn("[SLOW_QUERY] elapsed={}ms sql={} traceId={}",
                elapsed,
                si.getSqlWithValues(),    // 바인딩 파라미터 포함 SQL
                MDC.get("traceId"));
        }
    }
}
```

---

## 7. Micrometer로 Slow Query 메트릭화

```java
@Component
@RequiredArgsConstructor
public class QueryMetrics {

    private final MeterRegistry meterRegistry;

    // 쿼리 실행 시간 히스토그램
    public Timer.Sample startTimer() {
        return Timer.start(meterRegistry);
    }

    public void recordQueryTime(Timer.Sample sample, String queryName, long elapsedMs) {
        sample.stop(Timer.builder("jpa.query.duration")
            .tag("query", queryName)
            .tag("slow", String.valueOf(elapsedMs >= 1000))
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry));

        // 1초 이상이면 카운터 추가
        if (elapsedMs >= 1000) {
            meterRegistry.counter("jpa.query.slow.total",
                "query", queryName).increment();
        }
    }
}
```

```yaml
# Grafana 알람: slow query 발생 시 알람
# PromQL:
# rate(jpa_query_slow_total[5m]) > 0
# → "최근 5분 내 Slow Query 발생 시 알람"
```

---

## 8. logback-spring.xml — 감사 로그·Slow Query 분리

```xml
<!-- logback-spring.xml -->
<configuration>

    <!-- 메인 로그: JSON stdout (K8s/Docker 수집용) -->
    <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>email</includeMdcKeyName>
        </encoder>
    </appender>

    <!-- Slow Query 전용 파일 로거 -->
    <appender name="SLOW_QUERY_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/slow-query.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/slow-query.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>14</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <!-- slow-query 로거 분리 -->
    <logger name="slow-query" level="WARN" additivity="false">
        <appender-ref ref="SLOW_QUERY_FILE"/>
        <appender-ref ref="JSON_STDOUT"/>   <!-- 동시에 메인 로그에도 -->
    </logger>

    <root level="INFO">
        <appender-ref ref="JSON_STDOUT"/>
    </root>

</configuration>
```

---

## 9. Prometheus 스크레이프 설정

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
  metrics:
    tags:
      application: ${spring.application.name}    # 모든 메트릭에 앱명 태그 자동 추가
    distribution:
      percentiles-histogram:
        jpa.query.duration: true    # 히스토그램 (P50/P95/P99 계산 가능)
      percentiles:
        jpa.query.duration: 0.5, 0.95, 0.99
```

```yaml
# prometheus.yml — 스크레이프 설정
scrape_configs:
  - job_name: 'spring-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['myapp:8080']
    scrape_interval: 15s

# K8s 환경 (ServiceMonitor — Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```

### Grafana PromQL — AOP 감사 로그·Slow Query 대시보드

```promql
# 패널 1: 초당 API 요청 수 (Controller 기준)
rate(http_server_requests_seconds_count{application="myapp"}[1m])

# 패널 2: API 응답시간 P95
histogram_quantile(0.95,
  rate(http_server_requests_seconds_bucket{application="myapp"}[5m]))

# 패널 3: Slow Query 발생 추이
rate(jpa_query_slow_total{application="myapp"}[5m])

# 패널 4: 쿼리 실행시간 P99 (히스토그램)
histogram_quantile(0.99,
  rate(jpa_query_duration_seconds_bucket[5m]))

# 패널 5: 에러율
rate(http_server_requests_seconds_count{status=~"5.."}[1m])
/ rate(http_server_requests_seconds_count[1m])
```

### 알람 룰

```yaml
# alerts.yml (Prometheus Alerting Rules)
groups:
  - name: app-alerts
    rules:
      - alert: SlowQueryDetected
        expr: rate(jpa_query_slow_total[5m]) > 0
        for: 0m    # 즉시 알람
        labels:
          severity: warning
        annotations:
          summary: "Slow Query 발생: {{ $labels.application }}"
          description: "1초 이상 쿼리가 감지됐습니다. 즉시 확인이 필요합니다."

      - alert: HighApiErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
          / rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "API 에러율 5% 초과: {{ $labels.application }}"
```

---

## 10. 관련
- [[Logging]] — SLF4J + Logback 기초, MDC, 로그 레벨
- [[Metrics]] — Micrometer, 커스텀 메트릭, PromQL
- [[PLG-Stack]] — 로그 수집·조회 파이프라인
- [[../testing/Unit-Integration-Testing]] — @WithMockJwtUser로 감사 로그 테스트
- [[../resilience/Resilience4j]] — Circuit Breaker Prometheus 연동
- [[../Security-Checklist]] — 감사 로그와 보안 체크리스트 연계
