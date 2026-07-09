---
tags:
  - database
  - data-access
  - moc
  - index
created: 2026-06-15
---

# 🗄️ Database / Data Access MOC

> Java/Spring 환경에서 많이 쓰는 **데이터 접근(persistence) 기술** 정리.
> 크게 **동기(Blocking)** 와 **비동기(Reactive)** 로 나뉜다.

## 동기 (Blocking, JDBC 기반)
- [[JDBC]] — 가장 저수준 표준 API (모든 것의 토대)
- [[JdbcTemplate]] — Spring의 JDBC 추상화 / Spring Data JDBC
- [[MyBatis]] — SQL Mapper (SQL을 직접 작성)
- [[JPA-Hibernate]] — ORM 표준 (객체↔테이블 매핑)
- [[QueryDSL]] — 타입세이프 동적 쿼리 (JPA/SQL 보조)

## 비동기 (Reactive, Non-blocking)
- [[R2DBC]] — Reactive Relational DB 연결 (WebFlux와 결합)

## 개념/비교
- [[Sync-vs-Async-DataAccess]] — 동기 vs 비동기 선택 기준 ⭐
- [[Connection-Pool]] — 커넥션 풀(HikariCP vs Agroal 등) + 풀 크기 산정
- [[PgBouncer]] — 서버사이드 풀러(다중화) & 도입 분석
- [[RDS-vs-Self-Hosted]] — 매니지드 DB(RDS/Aurora) vs VM 직접 구축

## 운영/동시성
- [[Transaction-Management]] — `@Transactional` 전파(7종)·격리 수준·rollback 규칙·readOnly·프록시 함정 ⭐
- [[DB-Migration]] — 스키마 버전 관리(Flyway/Liquibase), expand-contract 무중단 마이그레이션
- [[Distributed-Lock]] — 분산 락: 낙관/비관, DB advisory lock, Redis(Redisson), Redlock 논란

## 인프라/테스트 연계
- [[Load-Testing|부하 테스트 전략]] — 로컬 vs 클라우드

---

## 🧭 한눈에 비교

| 기술 | 방식 | 추상화 수준 | SQL 작성 | 비고 |
|------|------|------------|----------|------|
| **JDBC** | 동기 | 최저 | 직접 | 표준, 보일러플레이트 多 |
| **JdbcTemplate** | 동기 | 낮음 | 직접 | JDBC 보일러플레이트 제거 |
| **Spring Data JDBC** | 동기 | 중간 | 일부 자동 | 단순 매핑, 지연로딩 없음 |
| **MyBatis** | 동기 | 중간 | 직접(XML/애너) | SQL 완전 제어 |
| **JPA/Hibernate** | 동기 | 높음 | 자동(JPQL) | ORM, 생산성 ↑, 학습난도 ↑ |
| **R2DBC** | 비동기 | 중간 | 직접 | 논블로킹, WebFlux |

---

## 🔑 선택 가이드 (요약)
- **복잡한 도메인 + 생산성** → [[JPA-Hibernate]] (+ [[QueryDSL]])
- **복잡한 SQL/튜닝/레거시 DB** → [[MyBatis]]
- **단순/경량/배치** → [[JdbcTemplate]] or Spring Data JDBC
- **고동시성 I/O 바운드 + 리액티브 스택** → [[R2DBC]]
- 자세한 기준은 [[Sync-vs-Async-DataAccess]] 참고.
