---
tags:
  - pattern
  - architecture
  - inbox
  - idempotency
  - messaging
  - distributed-transaction
created: 2026-06-17
---

# Idempotent Consumer / Inbox Pattern (멱등 컨슈머 · 인박스)

> [!summary] 한 줄 요약
> at-least-once 전달로 **중복 수신**되는 메시지를, 처리한 **메시지 ID를 inbox 테이블에 기록**하고 그 기록을 **비즈니스 처리와 같은 로컬 트랜잭션**으로 묶어 흡수하는 패턴. [[Outbox-Pattern]](발행 원자성)의 짝 — **수신 멱등성**을 담당한다.

---

## 1. 어떤 문제를 푸는가 — 중복 수신

```
[문제] 메시지 브로커는 사실상 at-least-once → 같은 메시지가 두 번 온다.
   ├─ Kafka rebalance: offset 커밋 전 파티션 이동 → 재처리
   ├─ 컨슈머 재시도 / DLT 재투입
   └─ producer 재전송 (ack 유실로 같은 메시지 재발행)

   중복을 그대로 처리하면 ⇒ 이중 결제, 재고 이중 차감, 포인트 이중 적립
```

→ **Inbox = 컨슈머 측 중복 방어.** [[Outbox-Pattern]]이 producer 측 *발행 원자성*을 책임진다면, Inbox는 consumer 측 *수신 멱등성*을 책임진다. [[Kafka]]·[[Saga-Pattern]]이 반복해서 말하는 "**멱등 처리 필수**"의 실제 구현이다.

---

## 2. 동작 원리

```
① 메시지 수신 (messageId 동반)
② inbox 테이블에 messageId 있나? ── 있으면 → skip (이미 처리됨, ack만)
                                └ 없으면 ↓
③ 한 로컬 트랜잭션으로:
     ├─ 비즈니스 변경 (orders / payment ...)
     └─ inbox INSERT (messageId)          ← 원자적 커밋
④ ack (offset commit)
```

> [!important] 핵심: 처리와 기록이 한 트랜잭션이어야 한다
> 비즈니스 처리와 messageId 기록이 **다른 트랜잭션**이면, 처리 성공 후 기록 전에 크래시 시 messageId가 없어 **재처리**된다. 둘을 같은 `@Transactional`로 묶어야 "처리했으면 반드시 기록됐다"가 성립한다 — Outbox가 "변경했으면 반드시 발행 레코드가 있다"를 보장하는 것과 정확히 대칭.

---

## 3. 구현 방식

### (a) Inbox 테이블 + 같은 트랜잭션 (가장 일반적)

```sql
CREATE TABLE processed_messages (
    message_id   VARCHAR(64) PRIMARY KEY,   -- 멱등성 키
    consumer     VARCHAR(64) NOT NULL,      -- group/서비스별 분리 시
    processed_at TIMESTAMP   NOT NULL DEFAULT now()
);
```

INSERT를 비즈니스 변경과 한 트랜잭션에. 중복이면 **PK 충돌**로 잡아 skip.

### (b) DB unique constraint / upsert — 자연키로 멱등

```sql
-- 별도 inbox 없이 비즈니스 테이블 자체로 멱등
CREATE TABLE payment (
    id       BIGINT PRIMARY KEY,
    order_id BIGINT UNIQUE,         -- 같은 주문 결제는 한 번만
    amount   NUMERIC
);
INSERT INTO payment(order_id, amount) VALUES (?, ?)
ON CONFLICT (order_id) DO NOTHING;  -- 중복이면 무시
```

### (c) 자연 멱등 연산 — 연산 자체가 여러 번 해도 동일

```sql
UPDATE orders SET status = 'PAID' WHERE id = ? AND status = 'PENDING';
-- 상태 머신/절대값 SET은 N번 적용해도 결과 동일 → inbox 불필요
-- 주의: 증분(balance = balance + 10)은 멱등 아님 → (a)/(b) 필요
```

### (d) Redis 기반 dedup (SETNX + TTL) — 빠르지만 트레이드오프

```java
Boolean first = redis.opsForValue()
    .setIfAbsent("inbox:" + messageId, "1", Duration.ofHours(6)); // SETNX + TTL
if (Boolean.FALSE.equals(first)) return;   // 이미 처리 → skip
```

| 방식 | 정합성 | 속도 | 비고 |
|------|--------|------|------|
| (a) Inbox 테이블 | 강함 (DB 트랜잭션과 원자적) | 보통 | 기본 권장 |
| (b) unique/upsert | 강함 | 빠름 | 자연키 있을 때 최선 |
| (c) 자연 멱등 | 강함 | 가장 빠름 | 가능하면 1순위 |
| (d) Redis SETNX | 약함 (영속성·정합성 ↓) | 매우 빠름 | 윈도우 한계, 유실 시 통과 |

> Redis는 비즈니스 트랜잭션과 원자적이지 않다 → "Redis는 기록됐는데 DB 커밋 실패"가 가능. 강한 정합성이 필요하면 DB inbox를, 초고처리량·중복이 드물면 Redis를 고려.

---

## 4. 멱등성 키 선택

| 키 | 설명 | 적합 |
|----|------|------|
| **messageId** (권장) | producer가 부여한 고유 ID. 브로커 헤더로 전달 | 범용 — 어떤 메시지든 |
| **비즈니스 자연키** (orderId 등) | 도메인 식별자. (b) upsert와 궁합 | "주문당 1회" 같은 도메인 멱등 |
| **content hash** | payload 해시 | ID 부재 시 차선 (의도된 동일 재전송과 구분 못 함) |

> `correlationId`와 혼동 금지. **correlationId**는 한 워크플로우(주문 처리 전체)를 묶는 추적 ID로 여러 메시지가 공유한다. **멱등성 키**는 *개별 메시지 1건*을 식별한다 — 중복 판정 기준이 다르다.

---

## 5. Inbox 테이블 정리 (retention)

inbox는 그대로 두면 **무한 증가**한다. 멱등 윈도우(=얼마나 오래된 중복까지 막을지)를 정해 정리한다.

```sql
-- 오래된 처리 기록 삭제 (멱등 윈도우 = 7일 가정)
DELETE FROM processed_messages WHERE processed_at < now() - INTERVAL '7 days';
```

- **TTL / 파티셔닝**: 날짜 파티션 후 오래된 파티션 DROP(삭제보다 저렴).
- **트레이드오프**: 윈도우가 **짧으면** 늦게 도착한 중복(지연된 재전송)이 통과해 이중 처리, **길면** 테이블·인덱스 비대 → 조회 비용 증가. 브로커의 최대 재전송 지연보다 넉넉히 잡는다.

---

## 6. Spring 구현 예시

```java
@Component
@RequiredArgsConstructor
public class PaymentConsumer {
    private final InboxRepository inboxRepository;
    private final PaymentService paymentService;
    private final OutboxRepository outboxRepository;   // Outbox와 결합

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    @Transactional                                     // ⭐ 체크·처리·기록·발행이 한 트랜잭션
    public void consume(OrderPlacedEvent event,
                        @Header("messageId") String messageId,
                        Acknowledgment ack) {
        if (inboxRepository.existsById(messageId)) {   // ① 이미 처리?
            ack.acknowledge();                         //    → skip, ack만
            return;
        }
        paymentService.charge(event.orderId());        // ② 비즈니스 처리
        inboxRepository.save(new InboxMessage(messageId));  // ③ 처리 기록 (PK 충돌 시 중복)
        outboxRepository.save(OutboxMessage.of(        // ④ 후속 이벤트 발행도 같은 트랜잭션
            "PaymentCompleted", event.orderId().toString(), /* payload */ "..."));
        ack.acknowledge();                             // ⑤ 커밋 후 offset ack
    }
}
```

> `existsById` 체크와 `save`는 동시성 환경에서 **PK 충돌(unique violation)로 한 번 더 방어**된다. 체크를 통과한 두 트랜잭션이 동시에 INSERT하면 하나는 충돌 → 롤백/skip. 즉 select 체크는 최적화, **PK 제약이 최종 보루**다.

**End-to-end (Inbox + Outbox)**: 메시지를 **Inbox로 받아 멱등하게 처리**하고, 그 결과 이벤트를 **Outbox로 원자적으로 발행**한다. 수신·처리·재발행이 한 트랜잭션이 되어 단계 간 신뢰 사슬이 끊기지 않는다.

```
[브로커] ─► Inbox(수신 멱등) ─► 비즈니스 처리 ─► Outbox(발행 원자성) ─► [브로커]
            └──────────────── 한 로컬 트랜잭션 ────────────────┘
```

---

## 7. Outbox vs Inbox 대조

| | [[Outbox-Pattern]] | Inbox (이 문서) |
|---|---|---|
| 위치 | **Producer** 측 | **Consumer** 측 |
| 보장 | 발행 **원자성** (변경=발행) | 수신 **멱등성** (중복 흡수) |
| 푸는 문제 | dual write (DB·브로커 불일치) | 중복 수신 (이중 처리) |
| 핵심 테이블 | outbox (보낼 메시지) | inbox / processed_messages (받은 메시지 ID) |
| 트랜잭션 | 비즈니스 변경 + outbox INSERT | 비즈니스 처리 + inbox INSERT |
| 전달 시맨틱 | at-least-once **발행** | at-least-once를 **effectively-once로** 변환 |

> 둘 다 쓰면 EOS(exactly-once)에 가까운 **effectively-once**가 된다. 브로커 자체의 exactly-once(Kafka EOS)와 달리, 애플리케이션 레벨에서 DB 트랜잭션만으로 달성하므로 이기종 브로커·재시도에도 견고하다.

---

## 8. 주의점 / 안티패턴

- ❌ **처리와 기록이 다른 트랜잭션** → 처리 후 기록 전 크래시 시 재처리. 반드시 한 `@Transactional`.
- ❌ **Redis-only dedup에 강정합성 의존** → Redis flush/장애 시 키 유실 → 중복 통과. 결제 등 정합성 중요 도메인은 DB inbox.
- ❌ **멱등 윈도우가 짧음** → 늦게 도착한 중복이 통과(5절). 브로커 최대 재전송 지연보다 길게.
- ❌ **select 체크만 믿고 PK 제약 누락** → 동시 중복이 둘 다 통과. **unique/PK가 최종 보루**.
- ❌ **증분 연산을 자연 멱등으로 착각** (`balance + 10`) → (a)/(b)로 보강.
- ⚠️ ack를 처리·기록보다 **먼저** 하면 at-most-once로 바뀌어 유실. 커밋 후 ack.

---

## 9. 관련

- [[Outbox-Pattern]] — 짝이 되는 발행 원자성 패턴 (producer 측)
- [[Kafka]] — at-least-once·rebalance 중복의 출처, "멱등 처리 필수"
- [[Saga-Pattern]] — 보상/재시도로 인한 중복 단계 실행 방어
- [[MQ-Concepts]] — 전달 보장(at-least/at-most/exactly-once)
- [[MSA]] — 서비스 간 비동기 통신의 신뢰성 기반

## 참고
- Chris Richardson, *Microservices Patterns* — Idempotent Consumer
- microservices.io/patterns/communication-style/idempotent-consumer.html
