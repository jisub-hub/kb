---
tags:
  - pattern
  - moc
  - index
created: 2026-06-15
---

# 🗂️ Pattern MOC (Map of Content)

> 아키텍처 & 디자인 패턴 정리 노트 인덱스.

## 아키텍처 패턴
- [[CQRS]] — 명령/조회 책임 분리
- [[DDD]] — 도메인 주도 설계
- [[Event-Sourcing]] — 이벤트로 상태 저장
- [[Hexagonal-Architecture]] — Ports & Adapters / Clean
- [[MSA]] — 마이크로서비스 아키텍처
- [[Eventual-Consistency]] — 결과적 일관성 (CAP/BASE, 일관성 모델, UX 대응)

## 디자인 패턴 (GoF)
- [[Creational-Patterns]] — 생성: Singleton·Factory Method·Abstract Factory·Builder·Prototype (+ Spring Bean)
- [[Structural-Patterns]] — 구조: Adapter·Proxy(AOP)·Decorator·Facade·Composite·Bridge·Flyweight
- [[Behavioral-Patterns]] — 행위: Strategy(DI)·Template Method·Observer·Command·CoR·State 등
- [[Adapter-Pattern]] — 어댑터 상세 (Hexagonal Ports & Adapters)
- [[Mediator-Pattern]] — 중재자 상세 (CQRS 핸들러 디스패치)

## 분산/메시징 패턴
- [[Saga-Pattern]] — 분산 트랜잭션 + 보상 (오케스트레이션/코레오그래피, 복구, 격리)
- [[Outbox-Pattern]] — 트랜잭셔널 아웃박스 (발행 원자성)
- [[Inbox-Pattern]] — 멱등 컨슈머/인박스 (수신 멱등성, Outbox의 짝)

## 데이터/영속성 패턴
- [[Repository-Pattern]] — 영속성 추상화

---

## 🔗 패턴 관계도

```
                ┌──────────────────┐
                │       DDD         │ (도메인 중심)
                └───────┬──────────┘
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   [Hexagonal]      [CQRS] ───── [Event-Sourcing]
        │               │               │
   [Repository]         └──── read ─────┘
                                │
                ┌───────────────┴───────────────┐
                ▼                                ▼
            [MSA] ──── 분산 트랜잭션 ──► [Saga] ── [Outbox]
```

- **DDD**가 토대 → 경계(Bounded Context)가 **MSA**의 서비스 분리 기준.
- **CQRS + Event Sourcing**은 한 쌍으로 자주 결합.
- **MSA**의 분산 일관성은 **Saga + Outbox**로 해결.

---

## 📌 작성 예정 / 확장 후보
- API Gateway / BFF
- Circuit Breaker (회복탄력성)
- Strangler Fig (레거시 점진적 대체)
