---
tags:
  - infra
  - architecture
  - msa
  - microservices
created: 2026-06-15
---

# MSA (Microservices Architecture) 환경

> [!summary] 한 줄 요약
> 단일 앱(모놀리스) 대신 **독립적으로 배포·확장 가능한 작은 서비스들의 집합**으로 구성하는 아키텍처. 서비스 간 통신, 분산 트랜잭션, 관찰성이 새로운 복잡성을 도입한다.

---

## 1. 모놀리스 vs MSA

| | 모놀리스 | MSA |
|---|---|---|
| 배포 단위 | 전체 앱 1개 | 서비스별 독립 배포 |
| 기술 스택 | 통일 | 서비스별 자유 (Polyglot) |
| 스케일링 | 전체 앱 스케일 아웃 | 병목 서비스만 스케일 아웃 |
| 장애 격리 | 하나 터지면 전체 영향 | 서비스 독립 → Circuit Breaker로 격리 |
| 데이터 관리 | 공유 DB | 서비스별 독립 DB |
| 복잡도 | 낮음 | 높음 (분산 시스템 문제) |
| 팀 구조 | 기능별 수직팀 | 서비스 오너십 팀 (Conway's Law) |

---

## 2. MSA 전형적 구성

```
클라이언트 (Web, Mobile, 3rd Party)
    ↓
[API Gateway]           ← 인증, Rate Limit, 라우팅 (Spring Cloud Gateway / Kong)
    ↓
[서비스 레이어]
  ┌─────────────────────────────────────────────────────┐
  │  [Order Service]    [User Service]    [Payment Service] │
  │   DB: PostgreSQL     DB: MySQL         DB: PostgreSQL   │
  └─────────────────────────────────────────────────────┘
    ↕ 동기(Feign/gRPC)         ↕ 비동기(Kafka/RabbitMQ)
  [Notification Service]    [Analytics Service]
    DB: MongoDB                DB: Elasticsearch
```

---

## 3. 서비스 분리 기준

**Domain-Driven Design (DDD) 기반:**
- **Bounded Context**: 업무 도메인의 경계 = 서비스 경계
- 각 서비스는 자신의 **데이터 완전 소유** (공유 DB 금지)

**실용적 기준:**
| 기준 | 설명 |
|---|---|
| **독립 배포** | 다른 팀에 영향 없이 배포 가능한가? |
| **독립 스케일** | 이 기능만 별도 스케일이 필요한가? |
| **팀 경계** | 팀별 오너십이 명확한가? |
| **데이터 경계** | DB 테이블을 독립적으로 소유 가능한가? |

> [!warning] 너무 작은 서비스는 오히려 독이 된다
> "Nano Service"는 서비스 간 호출 오버헤드가 크고 트랜잭션 관리가 지옥. 처음엔 모놀리스로 시작, 경계가 명확해지면 분리.

---

## 4. 서비스 간 통신

### 동기 통신 (REST / gRPC)

```
Order Service → (HTTP Feign Client) → Inventory Service
```

- 즉각 응답 필요할 때
- 단점: 호출 체인 길면 지연 누적, 하나 죽으면 연쇄 장애 → Circuit Breaker 필수

### 비동기 통신 (Event-Driven)

```
Order Service → [Kafka: order-placed event] → Notification Service
                                             → Analytics Service
```

- 느슨한 결합 (Loose coupling)
- 응답 기다릴 필요 없음 → 처리량 향상
- 단점: 최종 일관성(Eventual Consistency) → 즉시 반영 불가

---

## 5. 분산 트랜잭션 문제

**2PC(2-Phase Commit)는 MSA에서 사용 안 함** → 성능·가용성 저하

**해결책:**

### Saga Pattern
```
Order Service → 주문 생성
    ↓ event
Payment Service → 결제 처리
    ↓ event
Inventory Service → 재고 차감
    ↓ (실패 시)
Compensating Transaction (보상 트랜잭션) 역방향 실행
```

### Outbox Pattern
```
Order Service가 DB 트랜잭션 내에서:
  1. order 테이블에 저장
  2. outbox 테이블에 이벤트 저장  ← 같은 트랜잭션
  3. (별도 프로세스) outbox → Kafka 발행
→ DB 저장 성공 = 이벤트 발행 보장
```

---

## 6. 서비스 디스커버리

### k8s 환경
```
order-svc.production.svc.cluster.local
→ k8s DNS가 자동으로 Service IP 반환
→ Service가 실제 Pod IP로 로드밸런싱
```

### VM 환경 (Consul/Eureka)
```
서비스 시작 시 Consul/Eureka에 자신의 IP:Port 등록
클라이언트가 호출 전 Registry에서 대상 서비스 IP 조회
```

---

## 7. 핵심 관찰성 요소

분산 시스템에서 로그·메트릭만으로는 문제 추적 불가 → **분산 트레이싱** 필수.

```
요청 → API GW → Order Svc → Payment Svc → DB
        └ traceId: abc123
                   └ spanId: def456
                              └ spanId: ghi789
```

- **Trace ID**: 전체 요청 흐름의 고유 ID
- **Span ID**: 각 서비스 처리 단위
- 도구: Jaeger, Zipkin, OpenTelemetry

---

## 8. MSA 도입 시점

| 신호 | 내용 |
|---|---|
| 팀 간 배포 충돌 | 한 팀의 배포가 다른 팀 코드를 깨는 경우 |
| 특정 기능만 스케일 필요 | 결제만 부하 높아서 전체 앱을 스케일 아웃 |
| 기술 스택 다양화 필요 | AI 서비스는 Python, 메인은 Java |
| 팀 규모 50인 이상 | Conway's Law에 따라 서비스 경계 = 팀 경계 |

---

## 9. 관련
- [[3-Tier-Architecture]] · [[K8s-Architecture]] · [[MSA|Spring MSA]]
- [[Spring-Cloud-Gateway]] · [[../../programming/spring/resilience/_index|Resilience4j]]
