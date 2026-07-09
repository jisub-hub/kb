---
tags:
  - database
  - rdb
  - nosql
  - acid
  - fundamentals
created: 2026-06-16
---

# 데이터베이스 기초 — ACID, RDB vs NoSQL

> [!summary] 한 줄 요약
> **ACID**는 신뢰할 수 있는 트랜잭션의 4가지 속성. **RDB(RDBMS)**는 1970년대 관계형 모델로 탄생해 SQL·ACID·JOIN이 강점. **NoSQL**은 2000년대 대규모 웹 서비스가 RDB의 수평 확장 한계를 돌파하기 위해 등장했다.

---

## 1. RDB의 유래

```
1970 — Edgar Codd(IBM): "관계형 모델" 논문 발표
       핵심 아이디어: 데이터를 "테이블(관계)"로 표현,
                     집합론·술어논리 기반 쿼리
       기존 네트워크/계층형 DB의 물리적 구조 종속성 탈피

1974 — IBM System R: SQL 원형 개발 (SEQUEL)
1979 — Oracle v2: 최초 상업용 SQL RDBMS
1986 — SQL 표준화 (ANSI SQL)

현재 주요 RDB:
  PostgreSQL (오픈소스 표준), MySQL/MariaDB, Oracle, SQL Server
  SQLite (임베디드)

[RDB의 철학]
  - 데이터는 테이블·행·열로 표현 (정규화)
  - 외래키로 관계 표현 → JOIN으로 재결합
  - SQL: 선언형 쿼리 (WHAT만 기술, HOW는 옵티마이저가 결정)
  - ACID 트랜잭션으로 데이터 무결성 보장
```

---

## 2. ACID 원칙

```
A — Atomicity (원자성)
  트랜잭션 내 모든 연산은 "전부 성공" 또는 "전부 실패"
  
  예: 이체 트랜잭션
    BEGIN;
    UPDATE accounts SET balance = balance - 10000 WHERE id = 1;  ← 성공
    UPDATE accounts SET balance = balance + 10000 WHERE id = 2;  ← 장애 발생
    ROLLBACK;  ← 위 차감도 되돌림 (1번 계좌 잔고 복구)
  
  구현: Write-Ahead Log(WAL) — 커밋 전 로그 먼저 기록

──────────────────────────────────────────────────────────────
C — Consistency (일관성)
  트랜잭션 전후 데이터가 정의된 규칙(제약)을 항상 만족
  
  예: NOT NULL, UNIQUE, CHECK, FOREIGN KEY 제약이
      트랜잭션 완료 시점에 항상 유효해야 함
  
  주의: CAP 이론의 "C(Consistency)"와 다름
        ACID의 C = 비즈니스 규칙 준수
        CAP의 C = 모든 노드가 동일한 최신 데이터 읽기

──────────────────────────────────────────────────────────────
I — Isolation (격리성)
  동시 실행 중인 트랜잭션은 서로 간섭 없이 독립적으로 동작해야 함
  
  격리 수준 (낮을수록 빠름, 높을수록 안전):
  
  READ UNCOMMITTED
    다른 트랜잭션의 미커밋 데이터 읽기 가능
    ❌ Dirty Read 발생 (실제로 거의 안 씀)
  
  READ COMMITTED (PostgreSQL 기본값)
    커밋된 데이터만 읽음
    ❌ Non-Repeatable Read: 같은 SELECT를 두 번 실행하면 다른 결과
  
  REPEATABLE READ (MySQL InnoDB 기본값)
    트랜잭션 시작 시점의 스냅샷 기준으로 읽음
    ❌ Phantom Read: 다른 트랜잭션이 INSERT한 행이 집계에 포함
  
  SERIALIZABLE
    완전 직렬화 (가장 느림)
    ✅ 모든 이상 현상 방지
    PostgreSQL: SSI(Serializable Snapshot Isolation)로 구현
  
  구현: MVCC(Multi-Version Concurrency Control)
    → 쓰기 시 기존 행을 덮지 않고 새 버전 생성
    → 읽기는 트랜잭션 시작 시점의 버전 참조
    → 읽기-쓰기 경합 최소화

──────────────────────────────────────────────────────────────
D — Durability (지속성)
  커밋된 트랜잭션 결과는 시스템 장애(전원 차단, 크래시) 후에도 보존
  
  구현: WAL(Write-Ahead Log)
    - 데이터를 디스크에 쓰기 전에 WAL 로그 먼저 fsync
    - 크래시 후 WAL 재생으로 커밋된 트랜잭션 복구
```

---

## 3. 트랜잭션 격리 수준 (Isolation Level)

ACID의 I(Isolation)를 어디까지 보장할지의 단계. 격리를 높이면 **이상 현상이 줄지만 동시성·성능이 떨어진다**. SQL 표준은 4단계를 정의한다.

### 이상 현상 (Read Phenomena)

| 이상 현상 | 설명 | 예시 |
|----------|------|------|
| **Dirty Read** | 다른 트랜잭션의 **미커밋** 값을 읽음 | T2가 읽은 값을 T1이 ROLLBACK → 유령 데이터 |
| **Non-Repeatable Read** | 같은 행을 두 번 읽었는데 값이 다름 | 사이에 T2가 UPDATE+COMMIT |
| **Phantom Read** | 같은 조건의 **집합**을 두 번 읽었는데 행 수가 다름 | 사이에 T2가 조건에 맞는 행 INSERT |

> Dirty/Non-Repeatable은 **개별 행** 문제, Phantom은 **조건에 매칭되는 행 집합(범위)** 문제라는 점이 차이.

### 격리 수준 ↔ 허용 이상 현상

| 격리 수준 | Dirty Read | Non-Repeatable | Phantom | 비고 |
|----------|:---------:|:--------------:|:-------:|------|
| **READ UNCOMMITTED** | 허용 | 허용 | 허용 | 거의 안 씀 |
| **READ COMMITTED** | 방지 | 허용 | 허용 | **PostgreSQL 기본** |
| **REPEATABLE READ** | 방지 | 방지 | 허용* | **MySQL InnoDB 기본** |
| **SERIALIZABLE** | 방지 | 방지 | 방지 | 가장 안전·가장 느림 |

```sql
-- 격리 수준 설정 / 확인
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- 현재 트랜잭션
SHOW transaction_isolation;     -- PostgreSQL
SELECT @@transaction_isolation; -- MySQL
```

> *표준상 REPEATABLE READ는 Phantom을 허용하지만, **실제 구현은 다르다** — 아래 PostgreSQL/MySQL 차이 참고.

### MVCC 원리

```
MVCC (Multi-Version Concurrency Control)
  쓰기가 기존 행을 덮어쓰지 않고 "새 버전"을 만든다.
  각 행 버전에 생성/삭제 트랜잭션 ID(xmin/xmax)를 기록 → 스냅샷으로 가시성 판단.

  T1(읽기) ── 시작 시점 스냅샷 ──> 그 시점에 보이던 버전만 읽음
  T2(쓰기) ── 새 버전 생성 ──────> T1에게는 안 보임 (= 읽기/쓰기 비차단)

  효과: "읽기는 쓰기를 막지 않고, 쓰기는 읽기를 막지 않는다"
  비용: 죽은 버전(dead tuple) 누적 → 정리 필요
        PostgreSQL: VACUUM (autovacuum)
        MySQL InnoDB: undo log + purge 스레드
```

### PostgreSQL vs MySQL InnoDB 기본 동작 차이

```
[PostgreSQL] 기본 = READ COMMITTED
  - 매 SQL문마다 새 스냅샷 → 같은 트랜잭션 안에서도 다른 결과 가능
  - REPEATABLE READ: 트랜잭션 첫 쿼리 시점 스냅샷 고정,
                     쓰기 충돌 시 "could not serialize" 에러로 거부
  - SERIALIZABLE: SSI(Serializable Snapshot Isolation) — 직렬화 위반 탐지 시 abort

[MySQL InnoDB] 기본 = REPEATABLE READ
  - 일반 SELECT(스냅샷 읽기)는 트랜잭션 첫 읽기 시점 스냅샷 고정
  - Next-Key Lock(레코드 락 + 갭 락)으로 범위를 잠가
    표준과 달리 Phantom Read까지 상당 부분 방지
  - 단, 잠금 읽기(SELECT ... FOR UPDATE)는 최신 커밋 데이터를 봄
```

> [!tip] 같은 이름, 다른 동작
> "REPEATABLE READ"라도 PostgreSQL은 **쓰기 충돌을 에러로 거부**하고, MySQL은 **락으로 막는다**. 동일 격리 수준명이 DB마다 다르게 동작하므로, 격리 수준에 의존하는 로직은 대상 DB 기준으로 검증해야 한다.

### 데드락 — 발생과 예방

```
[발생] 두 트랜잭션이 서로가 가진 락을 순환 대기
  T1: lock A → (B 대기)
  T2: lock B → (A 대기)   ← 교착. DB가 한쪽을 victim으로 abort

[예방]
  ① 락 획득 순서 통일 — 항상 같은 순서(예: PK 오름차순)로 행 잠금
  ② 트랜잭션 짧게 — 잠금 보유 시간 최소화
  ③ 적절한 인덱스 — 풀스캔이 불필요한 갭 락을 넓게 잡는 것 방지
  ④ 재시도 — 데드락 victim은 애플리케이션에서 재시도 (멱등 보장 전제)
```

```sql
-- 데드락 진단
SHOW ENGINE INNODB STATUS;          -- MySQL: LATEST DETECTED DEADLOCK 섹션
SELECT * FROM pg_locks WHERE NOT granted;  -- PostgreSQL: 대기 중인 락
```

> [!note] Advisory Lock
> 행/테이블이 아닌 **애플리케이션이 정한 임의 키**에 거는 명시적 락. 분산 작업 직렬화·잡 중복 실행 방지 등에 쓴다. PostgreSQL `pg_advisory_lock(key)` / MySQL `GET_LOCK('name', timeout)`. DB 자원 무결성과 무관하게 "이 로직은 한 번에 하나만"을 강제할 때 유용.

---

## 4. NoSQL의 유래

```
2000년대 초: 웹 2.0 폭발적 성장
  - Google: GFS(2003), Bigtable(2006)
  - Amazon: Dynamo(2007)
  - Facebook: Cassandra(2008)

RDB의 한계:
  ① 수평 확장 어려움
     - JOIN, 트랜잭션 → 여러 노드에 데이터 분산하면 분산 트랜잭션 필요
     - 수직 확장(더 좋은 서버)이 한계에 도달

  ② 스키마 경직성
     - 빠른 기능 추가 시 ALTER TABLE이 프로덕션 다운타임 위험

  ③ CAP 트레이드오프
     - RDB: CP 우선 (일관성 + 파티션 내성, 가용성 희생)
     - 웹 서비스: AP 우선 필요 (항상 응답, 최종 일관성 허용)

"NoSQL" 용어: 2009년 Johan Oskarsson이 Twitter에서 처음 사용
의미: Not Only SQL (SQL 대체가 아닌, SQL이 항상 답은 아님)
```

---

## 5. RDB vs NoSQL 비교

```
[데이터 모델]
  RDB:   테이블 (행·열), 정규화, 외래키
  NoSQL: 문서(MongoDB), 키-값(Redis), 와이드컬럼(Cassandra), 그래프(Neo4j)

[스키마]
  RDB:   엄격한 스키마 (변경 = ALTER TABLE, 운영 위험)
  NoSQL: 유연/스키마리스 (필드 추가 자유, 하지만 앱 레벨 검증 필요)

[쿼리]
  RDB:   SQL (표준화, 강력한 JOIN, 집계)
  NoSQL: 각 DB 고유 쿼리 API (JOIN 대부분 없음)

[트랜잭션]
  RDB:   ACID (완전 지원)
  NoSQL: 제한적 (단일 문서/키는 원자적, 멀티 문서는 약함 또는 없음)
         MongoDB 4.0+: 멀티 문서 트랜잭션 (하지만 성능 저하)

[확장성]
  RDB:   수직 확장 주 (샤딩 어렵고 복잡)
  NoSQL: 수평 확장 설계 (Cassandra, MongoDB Sharding 내장)

[일관성]
  RDB:   강한 일관성 (ACID)
  NoSQL: 최종 일관성 (Eventually Consistent) 또는 조정 가능

[사용 사례]
  RDB:   금융, 주문, ERP — 무결성이 최우선
  NoSQL: 소셜, 콘텐츠, 실시간, 대용량 로그 — 유연성/확장이 우선
```

---

## 6. CAP 이론

```
CAP(Brewer's Theorem, 2000): 분산 시스템은 세 가지 중 동시에 두 가지만 보장 가능

Consistency (일관성)
  모든 노드가 항상 동일한 최신 데이터 반환

Availability (가용성)
  모든 요청이 응답을 받음 (오류 없이)

Partition Tolerance (파티션 내성)
  네트워크 분리(노드 간 메시지 손실)가 발생해도 시스템 동작

실제 분산 시스템에서 P는 선택이 아님(네트워크 장애는 불가피)
→ 실질적 선택: CP vs AP

CP 시스템 (일관성 우선, 파티션 시 가용성 포기)
  - PostgreSQL (Patroni), etcd, ZooKeeper, HBase
  - 파티션 발생 시 응답 거부 or 오류 반환

AP 시스템 (가용성 우선, 파티션 시 일관성 포기)
  - Cassandra, DynamoDB, CouchDB
  - 파티션 발생 시 오래된 데이터라도 응답, 나중에 동기화 (최종 일관성)

[선택 기준]
  금융·결제:  CP (돈이 맞아야 한다 > 서비스 중단)
  SNS 피드:   AP (1초 전 글이 안 보여도 서비스 중단보다 낫다)
  쇼핑 카트:  AP (아마존: 카트 데이터 불일치 < 고객 이탈)
```

---

## 7. 선택 결정 트리

```
① 트랜잭션 무결성이 필수인가?
   YES → RDB (PostgreSQL, MySQL)

② 스키마가 자주 바뀌는가? (빠른 반복 개발)
   YES → NoSQL 고려

③ 수평 확장이 필수인가? (TB 이상 쓰기)
   YES → NoSQL (Cassandra, MongoDB Sharding)

④ 전문 검색이 핵심인가?
   YES → Elasticsearch

⑤ 시계열 데이터인가?
   YES → TimescaleDB (PostgreSQL 기반) or InfluxDB

⑥ 키-값 캐시인가?
   YES → Redis

⑦ 그래프 관계인가?
   YES → Neo4j

⑧ 위 어느 것도 아니고 CRUD가 주라면?
   → 그냥 PostgreSQL (과도한 NoSQL 도입 금지)
```

---

## 8. 관련
- [[PostgreSQL-Internals]] — MVCC, WAL 상세, VACUUM
- [[MongoDB]] — 문서 지향 NoSQL, 애플리케이션 레벨 ACID
- [[Elasticsearch]] — 검색 특화 NoSQL
- [[TimescaleDB]] — 시계열 특화
- [[PostgreSQL-HA-Cluster]] — Patroni + etcd + HAProxy (CP 분산 DB)
