---
tags:
  - database
  - postgresql
  - timeseries
  - timescaledb
  - monitoring
created: 2026-06-16
---

# TimescaleDB — 시계열 데이터베이스

> [!summary] 한 줄 요약
> PostgreSQL 확장으로 시계열 데이터를 처리하는 **하이퍼테이블**을 제공한다. 자동 파티셔닝, 압축, 연속 집계로 메트릭·이벤트·IoT 데이터를 기존 SQL 문법으로 고성능 처리한다.

---

## 1. 왜 TimescaleDB인가

```
일반 PostgreSQL:
  시계열 데이터 1억 건 → 범위 쿼리 수십 초 (전체 테이블 스캔)

TimescaleDB 하이퍼테이블:
  동일 데이터 → 시간 청크로 자동 파티셔닝 → 범위 쿼리 밀리초
  + 압축(Columnar) → 저장소 90% 절감
  + 연속 집계(Materialized View 자동 갱신)
```

### 주요 용도
- 애플리케이션·서버 메트릭 수집 (Prometheus 대체 또는 장기 보관)
- 금융 틱 데이터, 거래 로그
- IoT 센서 데이터
- 클릭스트림, 사용자 이벤트 로그
- [[LLMOps|LLMOps]] 비용·토큰 사용량 시계열 추적

---

## 2. 설치

```sql
-- PostgreSQL 확장 활성화
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- 버전 확인
SELECT default_version FROM pg_available_extensions WHERE name = 'timescaledb';
```

```bash
# Docker
docker run -d --name timescale \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  timescale/timescaledb:latest-pg16

# Helm (K8s)
helm repo add timescale https://charts.timescale.com
helm install timescaledb timescale/timescaledb-single \
  --set patroni.bootstrap.dcs.postgresql.parameters.max_connections=200
```

---

## 3. 하이퍼테이블 생성

```sql
-- 일반 테이블 생성
CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    device_id   TEXT        NOT NULL,
    metric_name TEXT        NOT NULL,
    value       DOUBLE PRECISION,
    tags        JSONB
);

-- 하이퍼테이블로 변환 (time 컬럼 기준 자동 청크 파티셔닝)
SELECT create_hypertable('metrics', by_range('time'));
-- 기본 청크 크기: 7일 (데이터 볼륨에 따라 조정)
SELECT create_hypertable('metrics', by_range('time', INTERVAL '1 day'));

-- 공간 분할 추가 (선택 — 카디널리티 높은 컬럼)
SELECT create_hypertable('metrics', by_range('time'), by_hash('device_id', 4));

-- 청크 현황 확인
SELECT * FROM timescaledb_information.chunks
WHERE hypertable_name = 'metrics'
ORDER BY range_start DESC LIMIT 10;
```

---

## 4. 쿼리 패턴

```sql
-- ── 시간 범위 조회 (청크 프루닝 자동 적용) ─────────────────────────

-- 최근 1시간 CPU 메트릭
SELECT time, device_id, value
FROM metrics
WHERE metric_name = 'cpu_usage'
  AND time > now() - INTERVAL '1 hour'
ORDER BY time DESC;

-- ── 시간 버킷 집계 ────────────────────────────────────────────────

-- 5분 단위 평균 CPU 사용률
SELECT time_bucket('5 minutes', time) AS bucket,
       device_id,
       avg(value) AS avg_cpu,
       max(value) AS max_cpu,
       min(value) AS min_cpu
FROM metrics
WHERE metric_name = 'cpu_usage'
  AND time > now() - INTERVAL '24 hours'
GROUP BY bucket, device_id
ORDER BY bucket DESC;

-- ── 갭 채우기 (데이터 없는 구간을 NULL로) ───────────────────────────
SELECT time_bucket_gapfill('1 minute', time, now() - INTERVAL '1 hour', now()) AS bucket,
       device_id,
       avg(value) AS avg_value
FROM metrics
WHERE metric_name = 'latency_ms'
  AND time > now() - INTERVAL '1 hour'
GROUP BY bucket, device_id
ORDER BY bucket;

-- ── 이동 평균 (Window Function) ─────────────────────────────────
SELECT time,
       device_id,
       value,
       avg(value) OVER (
           PARTITION BY device_id
           ORDER BY time
           RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW
       ) AS moving_avg_5m
FROM metrics
WHERE metric_name = 'temperature'
  AND time > now() - INTERVAL '1 day';
```

---

## 5. 연속 집계 (Continuous Aggregates)

```sql
-- 1시간 집계 Materialized View — 자동 갱신
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
       device_id,
       metric_name,
       avg(value)  AS avg_val,
       max(value)  AS max_val,
       min(value)  AS min_val,
       count(*)    AS sample_count
FROM metrics
GROUP BY bucket, device_id, metric_name;

-- 자동 갱신 정책 (1시간마다 최근 2일치 갱신)
SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset => INTERVAL '2 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- 수동 갱신
CALL refresh_continuous_aggregate('metrics_hourly',
    now() - INTERVAL '24 hours', now());

-- 연속 집계 위에 다시 집계 (계층형 집계)
CREATE MATERIALIZED VIEW metrics_daily
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 day', bucket) AS day_bucket,
       device_id,
       metric_name,
       avg(avg_val) AS avg_val
FROM metrics_hourly
GROUP BY day_bucket, device_id, metric_name;
```

---

## 6. 압축 (Columnar Compression)

```sql
-- 압축 정책 활성화 (7일 이상 된 청크 자동 압축)
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id, metric_name',  -- 세그먼트 키
    timescaledb.compress_orderby   = 'time DESC'                  -- 정렬 키
);

SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- 압축 현황 확인
SELECT pg_size_pretty(before_compression_total_bytes) AS before,
       pg_size_pretty(after_compression_total_bytes)  AS after,
       round(after_compression_total_bytes::numeric
             / before_compression_total_bytes * 100, 1) AS ratio_pct
FROM hypertable_compression_stats('metrics');

-- 수동 압축
SELECT compress_chunk(c)
FROM show_chunks('metrics', older_than => INTERVAL '7 days') c;
```

---

## 7. 데이터 보존 정책 (Retention)

```sql
-- 90일 이상 된 데이터 자동 삭제
SELECT add_retention_policy('metrics', INTERVAL '90 days');

-- 보존 정책 확인
SELECT * FROM timescaledb_information.jobs
WHERE proc_name = 'policy_retention';

-- Tiering: 오래된 데이터를 S3로 이동 (Timescale Cloud)
SELECT add_tiering_policy('metrics', INTERVAL '30 days');
```

---

## 8. Spring Boot 연동

```groovy
// build.gradle — 별도 드라이버 불필요 (표준 PostgreSQL JDBC 사용)
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
runtimeOnly 'org.postgresql:postgresql'
```

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/metricsdb
    username: postgres
    password: password
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

```java
// 시계열 엔티티 — @Table에 하이퍼테이블 이름 지정
@Entity
@Table(name = "metrics")
public class Metric {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Instant time;

    private String deviceId;
    private String metricName;
    private Double value;
}

// 시간 범위 쿼리
@Repository
public interface MetricRepository extends JpaRepository<Metric, Long> {

    @Query("""
        SELECT m FROM Metric m
        WHERE m.metricName = :name
          AND m.time > :from
        ORDER BY m.time DESC
        """)
    List<Metric> findRecent(@Param("name") String name,
                            @Param("from") Instant from);

    // time_bucket 집계는 JPQL 미지원 → Native Query 사용
    @Query(value = """
        SELECT time_bucket('5 minutes', time) AS bucket,
               avg(value) AS avg_val
        FROM metrics
        WHERE metric_name = :name
          AND time > :from
        GROUP BY bucket
        ORDER BY bucket DESC
        """, nativeQuery = true)
    List<Object[]> findBucketedAvg(@Param("name") String name,
                                   @Param("from") Instant from);
}
```

---

## 9. 모니터링 쿼리

```sql
-- 하이퍼테이블 전체 크기
SELECT hypertable_name,
       pg_size_pretty(hypertable_size(hypertable_name::regclass)) AS total_size
FROM timescaledb_information.hypertables;

-- 청크별 크기 상위 10개
SELECT chunk_name, range_start, range_end,
       pg_size_pretty(total_bytes) AS size
FROM timescaledb_information.chunks
WHERE hypertable_name = 'metrics'
ORDER BY total_bytes DESC LIMIT 10;

-- 초당 삽입 통계 (pg_stat_user_tables 기반)
SELECT n_tup_ins, n_tup_upd, n_tup_del
FROM pg_stat_user_tables
WHERE relname = 'metrics';

-- 백그라운드 작업 상태 (압축·집계·보존 정책)
SELECT job_id, proc_name, schedule_interval,
       last_run_status, last_run_duration,
       next_start
FROM timescaledb_information.job_stats;
```

---

## 10. Grafana 연동

```
TimescaleDB → Grafana (PostgreSQL 데이터소스로 직접 연결)
```

```sql
-- Grafana 패널 쿼리 예시 (time_bucket으로 시각화)
SELECT
    time_bucket('$__interval', time) AS "time",
    avg(value) AS "CPU Usage"
FROM metrics
WHERE
    metric_name = 'cpu_usage' AND
    $__timeFilter(time)       -- Grafana 시간 범위 필터 자동 치환
GROUP BY 1
ORDER BY 1
```

---

## 11. 실전 경험 — IoT 시계열 데이터: Elasticsearch → TimescaleDB 전환

> **배경**: IoT 센서 데이터를 처음에는 Elasticsearch로 수집·저장. 이후 인계받을 사람을 고려한 운영 복잡도와 쿼리 복잡성 문제로 TimescaleDB로 전환.

```
[Elasticsearch로 시작했을 때 문제점]
  ❌ 운영 복잡도:
     - 클러스터 Shard 수·Replica 설정 지식 필요
     - ILM(Index Lifecycle) 설정 실수 → 디스크 풀 사고
     - 인계받는 사람이 ES 클러스터 운영 학습 필요
  ❌ 쿼리 복잡성:
     - 시간 범위 집계 → ES Aggregation DSL (JSON 중첩)
     - 비개발자나 DBA가 쿼리 작성 어려움
  ❌ 비용:
     - 역 인덱스 + _source 이중 저장 → 스토리지 비용 높음
     - JVM 힙 메모리 최소 1~2GB 필수

[TimescaleDB로 전환 후]
  ✅ SQL만 알면 모든 쿼리 가능 (time_bucket, avg, max...)
  ✅ PostgreSQL 위에서 동작 → 기존 JPA/JDBC 코드 재사용
  ✅ 인계받는 사람도 PostgreSQL 지식으로 운영 가능
  ✅ 압축(Columnar) → 저장 비용 대폭 절감
  ✅ Grafana에서 PostgreSQL 데이터소스로 직접 연결
  ✅ 예상보다 더 만족스러운 결과
```

### Elasticsearch가 적합한 IoT 시나리오 vs TimescaleDB가 적합한 경우

| 상황 | ES | TimescaleDB |
|------|-----|-------------|
| 센서 수치 시계열 (온도, 압력, 전류) | △ 가능하나 과도 | ✅ 최적 |
| 이벤트 로그 전문 검색 ("ERROR 포함") | ✅ | ❌ |
| 시간 범위 집계, 평균/최대/이동평균 | △ Aggregation DSL | ✅ SQL |
| 운영 인계 용이성 | ❌ 클러스터 학습 필요 | ✅ PostgreSQL 지식으로 OK |
| Grafana 연동 | ✅ (ES 데이터소스) | ✅ (PostgreSQL 데이터소스) |
| 이상 탐지·ML | ✅ X-Pack ML | ❌ 별도 도구 필요 |

> **결론**: IoT 수치 시계열 → TimescaleDB. 로그·이벤트 전문 검색 → Elasticsearch. 둘 다 필요하면 병행.

---

## 12. 관련
- [[PostgreSQL-Internals]] — 인덱스·EXPLAIN·autovacuum 기반 지식
- [[pgvector]] — 같은 PG 확장 패턴
- [[../observability/Prometheus-Grafana]] — Prometheus 메트릭을 TimescaleDB에 장기 보관 (Promscale)
- [[JPA-Hibernate]] — Spring Data 연동
