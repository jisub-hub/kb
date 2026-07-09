---
tags:
  - mq
  - messaging
  - event-driven
  - design
  - architecture
created: 2026-06-17
---

# 이벤트 메시지 설계 (Event Design)

> [!summary] 한 줄 요약
> 이벤트 드리븐의 "첫 갈림길" — **무엇을(Event vs Command)**, **얼마나(Notification/ECST/Fat-Thin)**, **어떤 봉투에(Envelope·correlationId)** 담을지를 결정하는 메시지 설계 의미론.

---

## 1. Event vs Command

| 구분 | Command | Event |
|------|---------|-------|
| 의미 | "해라" (명령, 의도) | "일어났다" (발생한 사실 통보) |
| 수신자 | **특정 1명** 지정 | 누구든 구독(0..N) |
| 거부 | **거부 가능** (검증 실패) | 거부 불가(이미 일어난 일) |
| 네이밍 | 명령형 `ChargePayment` | 과거형 `PaymentCharged` |
| 결합도 | 송신자가 수신자/의도를 앎 → **높음** | 발행자는 구독자를 모름 → **낮음** |

```java
// Command — 처리 요청, 실패할 수 있음
record ChargePayment(String orderId, long amount) {}

// Event — 이미 확정된 사실, 구독자는 자유롭게 반응
record PaymentCharged(String orderId, long amount, Instant chargedAt) {}
```

- Command를 토픽에 흘려보내면 사실상 "비동기 RPC". Event는 **사실의 브로드캐스트**.
- 결합도를 낮추려면 가능한 한 **Event(과거형)** 로 설계하고, 명령이 꼭 필요한 곳만 Command로.

---

## 2. 이벤트 스타일 3가지 (Martin Fowler)

- **Event Notification** — ID/최소 정보만 발행. 수신자가 필요 시 발행자 API를 **콜백 조회**.
  - 결합 낮음, 페이로드 작음 / **네트워크 왕복↑**, 발행자 가용성에 의존.
- **Event-Carried State Transfer (ECST)** — 이벤트에 **상태를 담아** 전송. 수신자는 **자체 복제본** 유지 → 조회 불필요.
  - 결합 낮음 + **가용성↑**(발행자 죽어도 동작) / 데이터 중복, **정합성(stale)** 부담.
- **Event Sourcing** — 상태 변화를 **이벤트 시퀀스로 저장**하고 재생. → [[Event-Sourcing]] 참조.

```json
// Event Notification — ID만, 상세는 조회
{ "type": "OrderPlaced", "orderId": "o-123" }

// ECST — 소비자가 바로 쓸 수 있게 상태 동봉
{ "type": "OrderPlaced", "orderId": "o-123",
  "customerId": "c-9", "items": [{"sku":"A","qty":2}], "total": 19800 }
```

| 기준 | Notification | ECST | Event Sourcing |
|------|------|------|------|
| 결합도 | 낮음(조회로 재결합) | **가장 낮음** | 낮음 |
| 가용성 | 발행자에 의존 | **독립적** | 독립적 |
| 페이로드 크기 | 작음 | 큼 | 이벤트 단위 |
| 정합성 | 조회 시점 최신 | **stale 가능** | 재생으로 일관 |

---

## 3. Fat event vs Thin event

페이로드에 상태를 **얼마나** 담을지의 축.

| | Thin event | Fat event |
|---|---|---|
| 내용 | ID + 최소 키 | 전체/풍부한 상태 |
| 소비자 | **조회 필요** | 독립적으로 처리 |
| 부담 | 발행자 부하·왕복 | **크기/버전/PII** 부담 |

- 선택 기준: 소비자 다수가 같은 데이터를 반복 조회 → **fat(ECST)** 가 유리. 소비자가 일부 필드만 쓰거나 데이터가 민감 → **thin**.
- 절충: 핵심 필드만 fat + 상세는 Claim-Check(5장)나 콜백.

---

## 4. 메시지 봉투(Envelope) 표준

페이로드(`data`)와 **메타데이터(envelope)** 를 분리한다. 라우팅·추적·멱등성은 메타데이터로 처리.

| 필드 | 의미 |
|------|------|
| `messageId` | 메시지 고유 ID → **멱등성 키** ([[MQ-Concepts]]) |
| `type` | 이벤트 타입 (`OrderPlaced`) |
| `version` | 스키마 버전 (6장) |
| `occurredAt` | 사실 발생 시각 |
| `source` | 발행 서비스/컨텍스트 |
| `correlationId` | **워크플로우 전체 추적 ID** — [[Saga-Pattern]] 전반에 전파, 동일 비즈니스 흐름의 모든 이벤트 공유 |
| `causationId` | **직접 원인이 된 이벤트의 messageId** (인과 사슬) |
| `traceId` | 분산 추적(OTel) 연계 |

```json
{
  "messageId": "evt-8f3c",
  "type": "PaymentCharged",
  "version": 2,
  "occurredAt": "2026-06-17T08:30:00Z",
  "source": "payment-service",
  "correlationId": "order-flow-123",
  "causationId": "evt-4a1b",
  "traceId": "00-1a2b3c...-01",
  "data": { "orderId": "o-123", "amount": 19800 }
}
```

- **correlationId vs causationId**: correlation은 "같은 흐름인가", causation은 "직전 원인이 무엇인가". 둘을 함께 기록하면 이벤트 인과 트리 복원 가능.
- 업계 표준 **CloudEvents**(`id`, `type`, `source`, `specversion`, `time`, `data`)를 envelope로 채택하면 상호운용성 확보.

---

## 5. Claim-Check 패턴

대용량 페이로드(이미지·첨부파일·큰 문서)는 메시지에 직접 싣지 않고 **object store(S3 등)** 에 저장 후 **참조(URL/key)만** 전송.

```
Producer ─► S3.put(blob) ─► key
         └─► publish { type, claimCheck: "s3://bucket/key" }
                                   │
Consumer ◄── consume ──────────────┘
         └─► S3.get(key) ─► 실제 blob 처리
```

- 동기: [[Kafka]] 기본 메시지 한계 **1MB**(`max.message.bytes`), 브로커 부하/복제 비용.
- 트레이드오프: 메시지는 가벼워지나 **외부 저장소 가용성·수명(GC)** 관리 필요. blob 만료가 소비보다 빠르면 안 됨.

---

## 6. 이벤트 버저닝 & 호환성

스키마는 시간이 지나며 진화한다. **소비자를 깨지 않고** 변경하는 것이 핵심.

| 변경 | 호환성 | 비고 |
|------|--------|------|
| 필드 추가(옵셔널) | **backward** | 구버전 소비자는 무시 |
| 필드 삭제 | **forward** | 신버전 소비자가 옛 이벤트 처리 |
| 타입/의미 변경 | **breaking** | 새 `type`/`version`으로 분리 |

- **version 필드**로 명시. 소비 시 버전별 **upcasting**(옛 이벤트 → 최신 구조 변환) 후 처리.
- **Tolerant Reader**: 소비자는 모르는 필드를 **무시**하고 필요한 필드만 읽는다(`FAIL_ON_UNKNOWN_PROPERTIES=false`).
- Schema Registry(Avro/Protobuf)로 호환성 규칙을 강제하면 안전.

```java
// upcasting: v1 이벤트를 최신 모델로 끌어올림
PaymentCharged upcast(JsonNode raw) {
    int v = raw.path("version").asInt(1);
    long amount = (v < 2) ? raw.get("price").asLong()   // v1: price
                          : raw.get("amount").asLong(); // v2: amount
    return new PaymentCharged(raw.get("orderId").asText(), amount);
}
```

---

## 7. 이벤트 설계 체크리스트

- [ ] **과거형 네이밍** — `OrderPlaced`, `PaymentCharged` (명령형은 Command로 분리).
- [ ] **자기완결성** — 소비자가 이해할 핵심 컨텍스트 포함(과한 thin/조회 의존 지양).
- [ ] **correlationId / causationId 전파** — 흐름 추적·디버깅·Saga 보상.
- [ ] **멱등성 키(messageId)** — at-least-once 전제, 중복 처리 안전 ([[MQ-Concepts]]).
- [ ] **너무 chatty하지 않게** — 의미 있는 도메인 사실 단위로(필드 단위 이벤트 남발 금지).
- [ ] **PII 최소화** — 민감정보는 thin + 조회, 또는 토큰화. 보존 로그에 남음을 유의.
- [ ] **버저닝 전략** — version 필드 + tolerant reader.

---

## 8. 관련

- [[Kafka]] · [[MQ-Concepts]] · [[Event-Sourcing]] · [[CQRS]] · [[Saga-Pattern]] · [[Outbox-Pattern]] · [[MSA]]
