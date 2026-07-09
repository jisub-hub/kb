---
tags:
  - database
  - pgbouncer
  - connection-pool
  - postgresql
  - scaling
created: 2026-06-15
---

# PgBouncer & 서버사이드 커넥션 풀러

> [!summary] 한 줄 요약
> **PgBouncer**는 PostgreSQL 앞단에 두는 경량 **서버사이드 커넥션 풀러**. 수천 개 클라이언트 연결을 소수의 실제 DB 커넥션으로 **다중화(multiplexing)** 한다. 앱-측 풀([[Connection-Pool]] HikariCP)만으로 해결 안 되는 **"인스턴스 폭증 → DB 커넥션 고갈"** 문제를 푼다.

---

## 1. 왜 필요한가 (도입 동기)
PostgreSQL은 **커넥션마다 별도 OS 프로세스 + 수~수십 MB 메모리**를 쓴다. 그래서 커넥션 수가 늘면:
- 메모리·컨텍스트 스위칭 폭증 → DB 전체 성능 저하.
- `max_connections`(기본 100) 한계에 부딪힘.

```
[문제] MSA/오토스케일 → 앱 인스턴스 50개 × HikariCP 20 = 1000 커넥션 요구
        하지만 PG는 100~500이 현실적 한계  → 고갈

[해결] 앱들 ──► PgBouncer (앞단) ──► PG에는 소수(예: 20~50) 커넥션만 유지
        클라이언트 1000개를 DB 커넥션 20개로 다중화
```

> [!note] 앱-측 풀 vs 서버-측 풀 — 둘은 다른 레이어
> - **앱-측(HikariCP)**: 앱↔(풀러/DB) 사이, 인스턴스별. 커넥션 획득 지연 제거.
> - **서버-측(PgBouncer)**: (앱들)↔DB 사이, 공유. **전체 DB 커넥션 총량 제한**.
> 둘은 함께 쓴다. 보통 PgBouncer 도입 시 **앱 풀 크기를 줄인다**.

---

## 2. PgBouncer 풀 모드 (가장 중요한 선택)
| 모드 | 커넥션 반환 시점 | 다중화 효율 | 제약 |
|------|-----------------|------------|------|
| **session** | 클라이언트 연결 종료 시 | 낮음 | 가장 안전(기본). 세션 기능 전부 사용 가능 |
| **transaction** | **트랜잭션 종료 시** | 높음 ⭐ | 세션 상태(prepared stmt, advisory lock, `SET`) 주의 |
| **statement** | 매 SQL 종료 시 | 최고 | 멀티-statement 트랜잭션 불가(가장 제한적) |

> [!tip] 실무 표준
> **transaction 모드**가 효율/호환의 스윗스팟. 단, 애플리케이션이 **세션 상태에 의존하지 않아야** 한다(서버사이드 prepared statement, `SET` 세션변수, advisory lock 등).

### Spring/JDBC 연동 주의 (transaction 모드)
```properties
# PostgreSQL JDBC 드라이버: 서버사이드 prepared statement 비활성/제한
prepareThreshold=0
# 또는 PgBouncer 1.21+ 는 prepared statement 지원 옵션 있음(max_prepared_statements)
```
- HikariCP `max-lifetime`/`idle-timeout`은 그대로 두되, **앱 풀 크기를 작게**.

---

## 3. 설정 예시 (pgbouncer.ini)
```ini
[databases]
mydb = host=10.0.0.5 port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
auth_type = scram-sha-256
pool_mode = transaction
max_client_conn = 1000        # 클라이언트(앱) 측 최대 연결
default_pool_size = 25        # DB로 나가는 실제 커넥션 수 (per db/user)
reserve_pool_size = 5
```
→ 앱은 `jdbc:postgresql://pgbouncer-host:6432/mydb` 로 접속.

---

## 4. 다른 풀러/프록시 비교 (트렌드)
| 도구 | 대상 | 특징 | 비고 |
|------|------|------|------|
| **PgBouncer** | PostgreSQL | 초경량, 단일 목적, 사실상 표준 | 가장 널리 사용 ⭐ |
| **Pgpool-II** | PostgreSQL | 풀링 + 부하분산 + 쿼리라우팅 + 복제관리 | 무겁고 복잡, 다기능 |
| **Supavisor** | PostgreSQL | Supabase 제작, **멀티테넌트·클라우드 네이티브**, Elixir | 대규모 SaaS용 신흥 |
| **RDS Proxy** | RDS/Aurora | AWS 매니지드 풀러, IAM·페일오버 통합 | [[RDS-vs-Self-Hosted]] |
| **ProxySQL** | MySQL | MySQL용 프록시/풀러/라우팅 | MySQL 진영 |
| **pgcat** | PostgreSQL | Rust, 샤딩·LB 지원 PgBouncer 대안 | 신흥 |

> 단순 커넥션 풀링만 필요하면 **PgBouncer**, 읽기분산/라우팅까지 필요하면 Pgpool-II/pgcat, AWS면 **RDS Proxy** 검토.

---

## 5. 도입 판단 체크리스트
✅ 도입 고려:
- 앱 인스턴스 수 × 앱 풀 크기 > DB `max_connections` 에 근접/초과
- 서버리스/오토스케일링으로 커넥션이 **폭발적·간헐적**으로 늘어남(Lambda 등)
- 짧은 커넥션을 매우 자주 맺고 끊는 워크로드
- DB 커넥션당 메모리 부담이 큰 PostgreSQL

❌ 불필요:
- 인스턴스 수가 적고 총 커넥션이 DB 한계 내에서 안정적
- 세션 상태에 강하게 의존하는데 호환성 검증이 어려움

## 6. 운영 주의
- PgBouncer 자체가 **단일 장애점** → 이중화(HAProxy/keepalived) 또는 사이드카 배치.
- `transaction` 모드 호환성(prepared statement, `SET`, advisory lock, LISTEN/NOTIFY) **반드시 테스트**.
- 모니터링: `SHOW POOLS;`, `SHOW STATS;` ([[Metrics]]로 노출).

## 7. 관련
- [[Connection-Pool]] · [[RDS-vs-Self-Hosted]] · [[JDBC]] · [[JPA-Hibernate]] · [[MSA]] · [[Kubernetes]]
