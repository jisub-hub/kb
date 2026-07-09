---
tags:
  - database
  - connection-pool
  - hikaricp
  - performance
created: 2026-06-15
---

# Connection Pool (커넥션 풀)

> [!summary] 한 줄 요약
> DB 커넥션 생성은 비싸므로, 미리 만들어 둔 커넥션을 **재사용**하는 풀. Spring Boot 기본은 **HikariCP**(가장 빠르고 안정적).

---

## 1. 왜 필요한가
- 커넥션 생성(TCP+인증+세션)은 **수십 ms** 소요 → 매 요청마다 만들면 치명적.
- 풀이 커넥션을 빌려주고/반납받아 재사용 → 지연 감소 + DB 보호.

## 2. HikariCP 설정 (Spring Boot)
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # 핵심 파라미터
      minimum-idle: 10
      connection-timeout: 3000     # 커넥션 못 얻으면 3초 후 예외
      idle-timeout: 600000
      max-lifetime: 1800000        # DB의 wait_timeout보다 짧게
```

## 2-1. 🆚 HikariCP 외 다른 커넥션 풀 (트렌드 비교)

> 결론 먼저: **2020년대 사실상 표준은 HikariCP**다. Spring Boot 2.0+ 기본값이며 성능·안정성·경량성에서 우위. 나머지는 대부분 레거시이거나 특정 런타임 전용.

| 풀 | 상태/트렌드 | 특징 | 언제 |
|----|------------|------|------|
| **HikariCP** | ⭐ 현행 표준 | 초경량·고성능, 바이트코드 최적화, Spring Boot 기본 | 기본 선택 |
| **Agroal** | 📈 부상 중 | **Quarkus 기본**, 트랜잭션 통합(XA) 우수, GraalVM 친화 | Quarkus/네이티브 |
| **Tomcat JDBC Pool** | 유지보수 | 톰캣 번들, 무난하나 Hikari보다 느림 | 레거시 톰캣 환경 |
| **Apache DBCP2** | 레거시 | 오래됨, 성능 낮음 | 신규 비권장 |
| **c3p0** | 사실상 EOL | 구식, 무거움 | 신규 사용 금지 |
| **Vibur-DBCP** | 소수 | 가볍고 빠름(틈새) | 특수 케이스 |
| **r2dbc-pool** | 리액티브 전용 | [[R2DBC]] 논블로킹 풀 | WebFlux/리액티브 |
| **Oracle UCP** | 벤더 | Oracle RAC 기능 통합 | Oracle 특화 |

> [!tip] 한 줄 가이드
> **JVM 동기 스택** → HikariCP. **Quarkus/네이티브** → Agroal. **리액티브** → r2dbc-pool. 그 외(DBCP2/c3p0/Tomcat)는 굳이 새로 도입할 이유 없음.

### 왜 HikariCP가 빠른가 (참고)
- ArrayList 대신 커스텀 `FastList`, 락 최소화, 바이트코드 경량화(Javassist), 불필요 기능 제거.
- "풀은 작고 빠르게, 나머지는 DB가 한다"는 철학.

---

## 3. 풀 크기 산정 — 크다고 좋은 게 아니다
> HikariCP 권장 공식 (시작점):
> **pool size ≈ Tn × (Cm − 1) + 1**  (Tn=코어수, Cm=동시 처리 쿼리수)

### 더 실무적인 산정 기준
- **출발점 경험칙**: `connections ≈ (코어 수 × 2) + 유효 스핀들 수` (PostgreSQL 위키 권장 계열).
  - 대부분 DB에서 **CPU 코어 × 2 ~ 4** 면 충분. 10~20을 넘기면 의심해볼 것.
- **Little's Law 기반 계산**:
  ```
  필요 커넥션 수 ≈ 목표 처리량(TPS) × 평균 쿼리 보유시간(s)
  예) 500 TPS × 0.02s(20ms) = 10 커넥션
  ```
- **작게 시작 → 부하 테스트로 조정**([[Load-Testing]]). 큰 풀은 거의 항상 안티패턴.
- 너무 크면 → DB 컨텍스트 스위칭/락 경합으로 **오히려 처리량 하락**(역U자 곡선).

### ⚠️ 가장 흔한 함정 — 앱 인스턴스 × 풀 크기
```
앱 20개 인스턴스 × 풀 30 = 600 커넥션 요구
하지만 PostgreSQL max_connections=100  → 커넥션 고갈/거부!
```
- **전체 합산이 DB `max_connections`를 넘으면 안 된다.**
- 인스턴스가 많아지는 MSA/오토스케일링([[Kubernetes]])에서 특히 위험.
- 각 PG 커넥션은 **별도 프로세스 + 수 MB 메모리** → 수백 개면 DB 자체가 부담.
- → 이 문제의 해법이 **서버사이드 풀러([[PgBouncer]])**. 앱-측 풀과 별개 레이어.

## 4. 흔한 문제
| 증상 | 원인/대응 |
|------|-----------|
| `Connection is not available` | 풀 고갈 → 긴 트랜잭션/누수, OSIV, 풀 크기 점검 |
| 커넥션 누수 | try-with-resources/`@Transactional` 경계 확인, `leak-detection-threshold` |
| 간헐적 끊김 | `max-lifetime` < DB `wait_timeout` 으로 설정 |

## 5. 비동기·가상스레드 환경의 풀 사이징

- [[R2DBC]]도 커넥션 풀 사용(`spring.r2dbc.pool`). 논블로킹이라 적은 커넥션으로 높은 동시성 처리.

### 5.1 가상스레드(Java 21+) 시대 — "풀만 늘리면 된다"는 함정 ⚠️

```
오해: VT는 블로킹을 값싸게 만드니, VT 수천 개 = 동시성 수천 → 풀도 수천?
현실: VT는 거의 무한히 만들 수 있지만, DB 커넥션(풀)은 여전히 한정된 자원
      → VT 수천 개가 모두 DB를 치면 풀에서 "대기"만 폭증 → 병목은 그대로 풀
```

- **VT는 스레드 병목을 없앨 뿐, 커넥션 병목은 그대로**다. 풀 크기는 여전히 **DB가 감당하는 동시 쿼리 수**(코어·디스크·`max_connections`)로 산정한다(3절 공식 유지).
- VT가 바꾸는 것: 풀 대기를 톰캣 스레드 고갈 없이 견딘다(블로킹 대기가 싸짐). 그래서 **풀을 무리하게 키울 필요가 없어진다** — 오히려 작은 풀 + VT 대기가 안전.
- **VT pinning 주의**: 커넥션 획득/JDBC 구간에 `synchronized`가 있으면 캐리어 스레드가 고정(pinning)되어 VT 이점이 사라진다 → 커넥션 풀·드라이버의 pinning 여부 확인. → [[VirtualThreads-vs-WebFlux]]

```
결론: VT 도입 = 풀 키우기 X. 풀 크기는 DB 기준 그대로,
      VT는 "그 풀을 기다리는 비용"을 싸게 만들 뿐.
```

## 6. 2계층 풀링 (앱-측 + 서버-측)
```
[App HikariCP] ──► [PgBouncer (서버사이드 풀러)] ──► [PostgreSQL]
  (인스턴스별)        (수천 클라이언트 → 소수 DB 커넥션 다중화)
```
- 앱 인스턴스가 많아 DB `max_connections`를 넘길 때 도입.

### 6.1 이중 풀링 함정 ⚠️

```
① pool_mode 충돌:
   PgBouncer transaction mode + JDBC prepared statement(서버 준비문) = 깨짐
   → 세션 상태(준비문·임시테이블·SET)가 트랜잭션마다 다른 백엔드로 가서 유실
   대응: prepareThreshold=0(JDBC) 또는 PgBouncer session mode(다중화 이득↓),
         또는 PgBouncer 측 준비문 지원 옵션(버전별)

② 사이징 상호작용:
   App 풀 × 앱 인스턴스 수 ≤ PgBouncer max_client_conn
   PgBouncer default_pool_size ≤ PostgreSQL max_connections
   → 두 풀을 독립적으로 키우면 어딘가에서 막힘. 계층별로 함께 계산.

③ 타임아웃 이중화:
   App 풀 타임아웃 + PgBouncer 대기 + DB statement_timeout이 겹침
   → 어느 계층에서 끊겼는지 진단 어려움. 계층별 메트릭 분리 수집.
```

- 자세히: [[PgBouncer]]

### 6.2 풀 고갈 진단 (어느 계층이 막혔나)
```
HikariCP: hikaricp_connections_pending(대기 큐), _acquire 시간, _timeout
PgBouncer: SHOW POOLS (cl_waiting, sv_active), maxwait
PostgreSQL: pg_stat_activity (active 연결 수, wait_event)
→ 대기 큐가 쌓이는 계층이 진짜 병목. 풀 크기 무작정 늘리기 전에 위치부터 특정.
```

## 7. 관련
- [[PgBouncer]] · [[RDS-vs-Self-Hosted]] · [[JDBC]] · [[JPA-Hibernate]] · [[R2DBC]] · [[Sync-vs-Async-DataAccess]] · [[Load-Testing]]
- [[Transaction-Management]] — REQUIRES_NEW 커넥션 2개 점유·긴 트랜잭션이 풀 사이징에 주는 영향
