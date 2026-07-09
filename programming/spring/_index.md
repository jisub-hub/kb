---
tags:
  - spring
  - moc
  - index
created: 2026-06-15
updated: 2026-06-15
---

# 🍃 Spring Backend Knowledge Base — Master MOC

> Spring 기반 백엔드 개발 전반의 지식 정리. 아래 영역별 인덱스로 진입.

## 📂 영역별 인덱스
| 영역 | 내용 |
|------|------|
| [[architecture/_index|🏛️ Architecture]] | CQRS, DDD, Event Sourcing, Saga, Outbox, Repository, Hexagonal, MSA |
| [[web/_index|🌐 Web & 통신]] | MVC vs WebFlux, Virtual Threads, REST, gRPC, Cloud Gateway |
| [[resilience/_index|🛡️ Resilience]] | Resilience4j (Circuit Breaker, Retry, Bulkhead...) |
| [[data/_index|🗄️ Data Access]] | JDBC, JPA, MyBatis, R2DBC, QueryDSL, 커넥션풀, PgBouncer, RDS |
| [[cache/_index|⚡ Cache]] | Redis, Valkey, 캐싱 전략 |
| [[messaging/_index|📨 Messaging]] | Kafka, RabbitMQ, 메시징 개념 |
| [[security/_index|🔐 Security]] | Spring Security, OAuth2/JWT |
| [[observability/_index|📊 Observability]] | Logging, Tracing, Metrics |
| [[deploy/_index|🚢 Deploy]] | Docker, Kubernetes, CI/CD |
| [[testing/_index|🧪 Testing]] | 부하 테스트 전략, Testcontainers |
| [[java/_index|☕ Java & JVM]] | GC 종류·선택, JVM 튜닝, 힙 설정 |
| [[Security-Checklist|🔒 Security Checklist]] | Spring Boot·DB·K8s 배포 전 보안 체크리스트 |

---

## 🧭 전체 그림 — Spring 기반 MSA
```
   Client
     │
     ▼
[Spring Cloud Gateway]  ── 인증(OAuth2/JWT), RateLimit, CircuitBreaker
     │
     ▼
[Service A] ──Feign + Resilience4j──► [Service B]
  │   │                                   │
  │   └─ Cache: Redis/Valkey              └─ Event: Kafka/RabbitMQ (Outbox/Saga)
  │
  └─ Data: JPA/MyBatis/R2DBC → (PgBouncer) → DB(RDS/Self-hosted)

전 구간 관측: Logging + Tracing + Metrics
배포: Docker → Kubernetes (CI/CD)
```

## 🌱 기초
- [[Spring-Core]] — IoC/DI, Bean·생명주기·스코프, 컴포넌트 스캔, AOP 개념
- [[Spring-Batch]] — 배치/스케줄링: Job/Step/Chunk, fault tolerance, @Scheduled vs Quartz vs Batch, ShedLock

## 🔑 핵심 의사결정 노트 (자주 찾는 비교)
- [[MVC-vs-WebFlux]] — 블로킹 vs 논블로킹
- [[VirtualThreads-vs-WebFlux]] — 가상스레드 vs 리액티브 성능 비교
- [[Sync-vs-Async-DataAccess]] — 동기/비동기 데이터 접근 선택
- [[RDS-vs-Self-Hosted]] — 매니지드 vs 직접 구축
- [[Connection-Pool]] — 풀 크기 산정 · HikariCP/Agroal
