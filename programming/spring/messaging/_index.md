---
tags:
  - mq
  - messaging
  - moc
  - index
created: 2026-06-15
---

# 📨 Message Queue / Streaming MOC

> 메시지 브로커 / 이벤트 스트리밍 정리.

- [[RabbitMQ]] — 전통적 메시지 브로커 (AMQP, 라우팅 강력) + 운영 심화(prefetch·Quorum Queue·Lazy·DLX 백오프)
- [[Kafka]] — 분산 이벤트 스트리밍 플랫폼. 팬아웃·멀티 pod 파티션 할당·완료 판단·운영/보안
- [[MQ-Concepts]] — 공통 개념: 전달보장, exactly-once 신화, 멱등성 구현, poison/재시도, DLQ, 스키마 진화, replay
- [[Event-Design]] — 이벤트 메시지 설계: Event vs Command, Notification/ECST, 봉투(correlationId/causationId), Claim-Check, 버저닝

## 관련 패턴
- [[Inbox-Pattern]] — 수신 멱등성 (중복 수신 방어)
- [[Outbox-Pattern]] — 발행 원자성 (dual-write 해결)
- [[Saga-Pattern]] · [[Event-Sourcing]] · [[CQRS]] · [[MSA]]

---

## 🧭 RabbitMQ vs Kafka 한눈에

| | RabbitMQ | Kafka |
|---|----------|-------|
| 모델 | 메시지 브로커(큐) | 분산 로그/스트리밍 |
| 프로토콜 | AMQP | 독자(TCP) |
| 라우팅 | 매우 강력(Exchange) | 토픽/파티션 단순 |
| 메시지 보존 | 소비 후 삭제(기본) | **로그 보존**(재소비 가능) |
| 처리량 | 중~높음 | **매우 높음** |
| 순서 보장 | 큐 단위 | 파티션 단위 |
| 대표 용도 | 작업 큐, RPC, 복잡 라우팅 | 이벤트 스트림, 로그, 대용량 파이프라인 |

> **복잡한 라우팅·작업 분배** → RabbitMQ / **대용량 이벤트·재처리·스트리밍** → Kafka
