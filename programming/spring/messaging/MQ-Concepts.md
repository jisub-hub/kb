---
tags:
  - mq
  - messaging
  - concept
created: 2026-06-15
---

# 메시징 공통 개념 (MQ Concepts)

> [!summary] 한 줄 요약
> RabbitMQ·Kafka 등 모든 메시징에 공통으로 적용되는 핵심 개념 — 전달 보장, 멱등성, 순서, DLQ, 백프레셔.

---

## 1. 전달 보장(Delivery Guarantee)
| 수준 | 의미 | 비고 |
|------|------|------|
| **at-most-once** | 최대 1회 (유실 가능) | 거의 안 씀 |
| **at-least-once** | 최소 1회 (**중복 가능**) | 기본/현실적 → 멱등성 필수 |
| **exactly-once** | 정확히 1회 | 비싸고 제약 많음(Kafka EOS 등) |

> 대부분 **at-least-once + 소비자 멱등성**으로 설계한다.

### 1.1 "exactly-once의 신화" ⚠️

> [!warning] exactly-once는 만능이 아니다
> Kafka EOS(transactions)·RabbitMQ 어떤 것도 **외부 부수효과까지 정확히 1회**를 보장하지 못한다.

```
Kafka EOS가 보장하는 범위:
  Kafka로 읽고 → 처리 → Kafka로 쓰기 (read-process-write)
  = "브로커 내부에서만" 원자적 정확히 1회

EOS가 보장 못 하는 것:
  메일 발송, 외부 결제 API 호출, 제3 DB 쓰기 등 외부 부수효과
  → 컨슈머가 커밋 직전 크래시하면 부수효과는 이미 나갔는데 재처리됨
```

**현실의 정답 = effectively-once = at-least-once + 멱등 소비자**. 즉 중복 전달을 인정하고, 컨슈머가 멱등하게 처리해 "결과적으로 한 번"을 만든다. exactly-once라는 단어에 의존해 멱등성을 생략하면 사고난다. → [[Inbox-Pattern]]

## 2. 멱등성(Idempotency) ⭐
같은 메시지를 여러 번 처리해도 결과가 같아야 한다. **at-least-once 환경의 필수 전제.**

### 2.1 구현 방식

```java
// (a) 처리한 메시지 ID 기록 → 중복이면 skip
//     ★ 핵심: ID 기록과 비즈니스 처리를 "같은 트랜잭션"으로 (Inbox 패턴)
@Transactional
public void handle(Message msg) {
    if (inboxRepository.existsById(msg.id())) return;   // 이미 처리 → skip
    process(msg);                                        // 비즈니스 변경
    inboxRepository.save(new Inbox(msg.id()));           // 같은 TX에서 기록
}
```
```java
// (b) DB unique constraint / upsert — 자연키로 멱등
//     INSERT ... ON CONFLICT DO NOTHING  (payment(order_id UNIQUE))

// (c) 자연 멱등 연산 — 상태 SET은 여러 번 해도 동일
//     UPDATE order SET status='PAID'   (O, 멱등)
//     UPDATE balance SET amount = amount - 100  (X, 비멱등 → 누적)
```

```
빠른 dedup이 필요하면 Redis SETNX + TTL:
  SET dedup:{messageId} 1 NX EX 3600  → 성공 시 첫 처리, 실패 시 중복
  단, 영속성 약함 + TTL 윈도우 한계 → 정합성 중요하면 DB Inbox 권장
```

### 2.2 멱등성 키 선택
| 키 | 출처 | 비고 |
|----|------|------|
| messageId | producer 부여(UUID) | 권장. 재전송돼도 동일 |
| 비즈니스 자연키 | orderId 등 | 자연 멱등 + unique 제약 |
| content hash | 페이로드 해시 | 키 없을 때 차선, 충돌 주의 |

> 구현 상세는 [[Inbox-Pattern]] (수신 멱등성) ↔ [[Outbox-Pattern]] (발행 원자성).

## 3. 순서(Ordering)
- **RabbitMQ**: 단일 큐 단위 순서 (멀티 컨슈머 시 깨질 수 있음).
- **Kafka**: **파티션 단위** 순서 → 순서가 중요한 메시지는 **같은 키**로 보내 같은 파티션에.

## 4. Poison Message & 재시도 전략

**Poison message** = 몇 번을 재시도해도 계속 실패하는 메시지(잘못된 포맷, 버그, 영구 불일치). 그냥 두면 **무한 재시도 루프**로 컨슈머가 막히고 뒤 메시지까지 적체된다.

```
재시도 전략 계층:
  ① 즉시 재시도 (1~3회)      — 일시적 네트워크 흔들림 대응
  ② 지연 재시도 (backoff)    — exponential: 1s → 2s → 4s ... (외부 서비스 회복 대기)
  ③ Retry 토픽 분리          — retry-5s / retry-1m 토픽으로 미뤄 메인 흐름 비차단
  ④ 한계 초과 → DLQ          — 격리 후 사람이 분석/재처리

중요: 재시도는 멱등 전제 (2절). 재시도마다 부수효과 반복되면 안 됨.
```

```java
// Spring Kafka: 비차단 재시도 + DLT (별도 retry 토픽으로 미룸)
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0),   // 1s,2s,4s
    dltStrategy = DltStrategy.FAIL_ON_ERROR)
@KafkaListener(topics = "order-events")
public void handle(OrderEvent e) { ... }   // 실패 → retry 토픽 → 한계 후 .DLT
```

> **재시도 가능/불가 구분이 핵심**: 일시적 오류(타임아웃·5xx)는 재시도, 영구 오류(검증 실패·4xx)는 즉시 DLQ. 모두 무한 재시도하면 안 된다.

## 5. DLQ (Dead Letter Queue)
- 처리 반복 실패/만료 메시지를 **격리**하는 큐/토픽.
- Kafka: Dead Letter Topic(`<topic>.DLT`), RabbitMQ: DLX(Dead Letter Exchange).
- **운영 필수**: DLQ 적재량 알림 + 원인 분석 + (수정 후) 재처리(replay) 경로. DLQ를 만들고 안 보면 무의미.

## 6. 백프레셔 & 컨슈머 랙
- 생산 속도 > 소비 속도면 적체 → **lag 모니터링** 필수.
- 대응: 컨슈머/파티션 증설(Kafka는 파티션 = 병렬성 상한), 배치 처리, 흐름 제어.
- **알림 임계**: lag 절대값보다 *증가 추세*와 *예상 소진 시간*을 본다. 지속 증가 = capacity 부족 신호.

## 7. 스키마 진화 (Schema Evolution)

프로듀서와 컨슈머는 **독립 배포**되므로 메시지 스키마 버전이 어긋난다. 깨지지 않으려면 호환성 규칙이 필요하다.

| 호환성 | 의미 | 허용 변경 |
|--------|------|----------|
| **Backward** | 새 컨슈머가 옛 메시지 읽기 | 필드 추가(기본값), 필드 삭제 |
| **Forward** | 옛 컨슈머가 새 메시지 읽기 | 필드 추가, optional 삭제 |
| **Full** | 양방향 | 위 교집합만 |

```
실무 원칙:
  - 필드 추가는 항상 optional + 기본값 (backward 안전)
  - 필드 삭제/타입 변경은 위험 → version 필드로 분기 또는 신규 토픽
  - 컨슈머는 tolerant reader: 모르는 필드는 무시
  - Schema Registry(Avro/Protobuf)로 호환성 강제 검증
```

> 메시지 봉투·버저닝 설계는 → [[Event-Design]]

## 8. Replay & 재처리
- Kafka는 로그를 보존하므로 **offset을 되감아 재소비** 가능(버그 수정 후 재처리, 신규 컨슈머 백필).
- ⚠️ replay 시 **부수효과 재발** 위험 → 멱등 소비자([[Inbox-Pattern]]) 전제. 또는 부수효과를 끈 채 상태만 재구성.
- RabbitMQ는 소비 후 삭제라 replay 불가 → 필요하면 Kafka 또는 [[Event-Sourcing]].

## 9. 발행 신뢰성
- DB 변경과 메시지 발행의 원자성(dual-write 문제) → [[Outbox-Pattern]].
- 발행 확인(Publisher Confirms / acks=all + min.insync.replicas).

## 10. 관련
- [[RabbitMQ]] · [[Kafka]] · [[Event-Design]] · [[Inbox-Pattern]] · [[Outbox-Pattern]] · [[Saga-Pattern]]
