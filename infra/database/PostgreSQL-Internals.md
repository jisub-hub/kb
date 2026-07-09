---
tags:
  - database
  - postgresql
  - performance
  - internals
created: 2026-06-16
---

# PostgreSQL Internals — 인덱스·실행계획·운영

> [!summary] 한 줄 요약
> PostgreSQL 내부 동작(인덱스 종류, EXPLAIN ANALYZE, VACUUM, autovacuum 튜닝)을 이해해야 쿼리 성능을 진단하고 운영 장애를 예방할 수 있다.

---

## 1. 인덱스 종류

| 타입 | 구조 | 언제 쓰는가 |
|------|------|------------|
| **B-Tree** | 균형 트리 | 기본값. 등치/범위/정렬 쿼리 모두 |
| **Hash** | 해시맵 | 등치(`=`)만, 범위 불가 — B-Tree로 대체 가능 |
| **GIN** | 역인덱스 | 배열(`@>`), JSONB(`?`), 전문검색(`@@`) |
| **GiST** | R-Tree 계열 | 지리(PostGIS), 범위 타입, 텍스트 유사도 |
| **BRIN** | 블록 범위 요약 | 물리적으로 정렬된 대용량 테이블(시계열, 로그) |
| **HNSW / IVFFlat** | 그래프/클러스터링 | 벡터 유사도 → [[pgvector]] |

```sql
-- 복합 인덱스: 쿼리 WHERE 절 컬럼 순서대로
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- user_id 단독 검색도 가능 (왼쪽 접두사 규칙)
-- status 단독 검색은 인덱스 미활용

-- 부분 인덱스: 특정 조건 행만 인덱싱 (크기↓, 속도↑)
CREATE INDEX idx_orders_pending ON orders(created_at)
    WHERE status = 'PENDING';

-- 표현식 인덱스
CREATE INDEX idx_users_lower_email ON users(lower(email));
-- WHERE lower(email) = 'user@example.com' 에서 사용
```

---

## 2. EXPLAIN ANALYZE 해석

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, m.name
FROM orders o
JOIN members m ON o.member_id = m.id
WHERE o.status = 'PENDING'
  AND o.created_at > now() - interval '7 days';
```

```
Nested Loop  (cost=0.43..128.50 rows=23 width=180)
             (actual time=0.052..2.341 rows=18 loops=1)
  Buffers: shared hit=95 read=12          ← 히트: 캐시, read: 디스크
  ->  Index Scan using idx_orders_pending on orders
        (cost=0.29..64.00 rows=23 width=120)
        (actual time=0.031..1.201 rows=18 loops=1)
        Index Cond: (created_at > ...)
  ->  Index Scan using members_pkey on members
        (cost=0.14..2.80 rows=1 width=60)
        (actual time=0.006..0.006 rows=1 loops=18)
Planning Time: 0.8 ms
Execution Time: 2.5 ms
```

### 핵심 포인트

| 항목 | 의미 | 주의 신호 |
|------|------|----------|
| `cost=시작..종료` | 플래너 추정 비용 (상대값) | 추정과 actual 차이 크면 통계 구식 |
| `rows=N` (actual) | 실제 처리 행 수 | 추정 rows와 10배 이상 차이 → `ANALYZE` 필요 |
| `loops=N` | 반복 횟수 | Nested Loop에서 rows×loops가 실제 I/O |
| `Buffers: read=N` | 디스크에서 읽은 블록 | 높으면 인덱스 누락 또는 캐시 부족 |
| `Seq Scan` | 전체 테이블 스캔 | 대용량 테이블에서 발생 시 인덱스 검토 |
| `Hash Join` vs `Nested Loop` | 대용량=Hash Join 유리 | Nested Loop: 내부 테이블 작을 때 |

```sql
-- 통계 갱신 (플래너가 rows 추정 개선)
ANALYZE orders;

-- 전체 DB 통계 + 데드튜플 청소
VACUUM ANALYZE;
```

---

## 3. VACUUM & Bloat

PostgreSQL은 UPDATE/DELETE 시 **이전 버전 행을 물리 삭제하지 않고** 데드 튜플로 남긴다(MVCC). VACUUM이 이를 회수한다.

```sql
-- 데드 튜플 현황 확인
SELECT schemaname, relname,
       n_dead_tup,
       n_live_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_ratio_pct,
       last_autovacuum,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 수동 VACUUM (dead_ratio > 20% 이상일 때)
VACUUM (VERBOSE, ANALYZE) orders;

-- 테이블 크기 vs 실제 데이터 크기 (bloat 확인)
SELECT pg_size_pretty(pg_total_relation_size('orders')) AS total,
       pg_size_pretty(pg_relation_size('orders'))       AS heap;
```

---

## 4. Autovacuum 튜닝

기본 autovacuum은 소형 DB에 맞춰져 있어 대용량 테이블에서 느리게 반응한다.

```sql
-- 전역 설정 확인
SHOW autovacuum_vacuum_scale_factor;     -- 기본 0.2 (20% 데드 시 발동)
SHOW autovacuum_vacuum_cost_delay;       -- 기본 2ms (I/O 스로틀)

-- 테이블별 오버라이드 (대용량 테이블 전용)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor  = 0.01,   -- 1% 데드 시 발동
    autovacuum_analyze_scale_factor = 0.005,  -- 0.5%에서 분석
    autovacuum_vacuum_cost_delay    = 0,       -- I/O 스로틀 제거
    autovacuum_vacuum_cost_limit    = 800      -- 기본 200에서 4배
);
```

```ini
# postgresql.conf — 전역 튜닝 (재시작 불필요, reload 적용)
autovacuum_max_workers      = 5      # 기본 3 → 대형 DB에서 증가
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 400
```

---

## 5. 파티셔닝

```sql
-- Range 파티셔닝 (시계열 데이터에 적합)
CREATE TABLE events (
    id         BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    payload    JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE events_2025_02 PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- 파티션 프루닝: WHERE created_at BETWEEN ... 이면 해당 파티션만 스캔
EXPLAIN SELECT * FROM events WHERE created_at = '2025-01-15';
-- → "Append" 아래 events_2025_01만 스캔 확인

-- List 파티셔닝 (enum 값으로 분할)
CREATE TABLE orders_by_region (...) PARTITION BY LIST (region);
CREATE TABLE orders_kr PARTITION OF orders_by_region FOR VALUES IN ('KR');
CREATE TABLE orders_us PARTITION OF orders_by_region FOR VALUES IN ('US');
```

---

## 6. 커넥션 관리

```ini
# postgresql.conf
max_connections = 200        # 과다 설정 금지 — 각 연결이 ~5MB 메모리 사용
# 연결 폭발 방지: 앱은 PgBouncer [[PgBouncer]] 경유 필수
```

```sql
-- 현재 연결 현황
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- 오래된 유휴 연결 강제 종료
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND query_start < now() - interval '10 minutes';
```

---

## 7. 관련
- [[pgvector]] · [[PgBouncer]] · [[Connection-Pool]]
- [[JPA-Hibernate]] · [[QueryDSL]]
