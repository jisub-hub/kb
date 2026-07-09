---
tags:
  - mq
  - kafka
  - streaming
  - event-driven
created: 2026-06-15
---

# Apache Kafka

> [!summary] 한 줄 요약
> 분산 **이벤트 스트리밍 플랫폼**. 메시지를 **append-only 로그**로 저장(보존)하여 **고처리량**·재소비·스트림 처리를 지원한다. 대용량 이벤트 파이프라인의 사실상 표준.

---

## 1. 사용처
- **이벤트 드리븐 아키텍처** ([[MSA]], [[Event-Sourcing]], [[CQRS]]).
- 대용량 로그/메트릭/IoT 수집 파이프라인.
- 스트림 처리(Kafka Streams, Flink), 실시간 분석.
- 서비스 간 비동기 통신 + **이벤트 재처리**(로그 보존 덕분).
- [[Saga-Pattern]] / [[Outbox-Pattern]] 의 이벤트 버스.

## 2. 핵심 개념
```
Producer ─► Topic(파티션 0,1,2 ...) ─► Consumer Group
                  │
            각 파티션 = 순서 보장된 append-only 로그
            메시지는 소비돼도 보존(retention 기간/크기까지)
```
- **Topic**: 메시지 카테고리. **Partition** 으로 분할(병렬성·확장).
- **Offset**: 파티션 내 위치. 컨슈머가 진행 위치 관리.
- **Consumer Group**: 같은 그룹 내 컨슈머가 파티션을 나눠 처리(스케일 아웃). 파티션 수 = 최대 병렬 컨슈머.
- **순서 보장**: **파티션 단위**만(전체 순서 X) → 같은 키는 같은 파티션.
- **보존(retention)**: 시간/크기 기준으로 보관 → **재소비 가능**(offset 리셋).

---

## 3. Spring for Apache Kafka 설정
```groovy
implementation 'org.springframework.kafka:spring-kafka'
```
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                          # 모든 ISR 확인 → 최고 내구성
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false          # 수동 커밋 권장
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    listener:
      ack-mode: manual                   # 처리 후 수동 커밋
```

## 4. Producer
```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public void publish(OrderCreatedEvent event) {
        kafkaTemplate.send("order-events", event.orderId().toString(), event)  // key→파티션 결정
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("발행 실패", ex);
                else log.info("발행 offset={}", result.getRecordMetadata().offset());
            });
    }
}
```
> **같은 key(orderId)** 는 항상 같은 파티션 → 해당 주문 이벤트의 **순서 보장**.

## 5. Consumer (수동 커밋)
```java
@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void consume(OrderCreatedEvent event, Acknowledgment ack) {
        try {
            // ... 멱등 처리 (offset 재처리·중복 대비) ...
            inventoryService.reserve(event.orderId());
            ack.acknowledge();                  // 처리 성공 후 offset 커밋
        } catch (Exception e) {
            log.error("처리 실패, 재시도/DLT", e);
            throw e;                            // ErrorHandler/DLT로 위임
        }
    }
}
```

### Dead Letter Topic + 재시도
```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);   // 실패 → <topic>.DLT
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3));  // 3회 재시도
}
```

## 6. 팬아웃 — 여러 서비스가 하나의 메시지를 본다

**갈림길은 `groupId`다.** 같은 토픽을 누가 어떻게 받는지가 consumer group으로 결정된다.

```
같은 group  → 파티션을 나눠 가짐 → 한 메시지는 그룹 내 "한 컨슈머만" (경쟁 소비/작업 분산)
다른 group  → 각 그룹이 같은 메시지를 "각자 모두" 받음 (fan-out/브로드캐스트)
```

```
order-events 토픽 (key=orderId)
  ├─ group=inventory  → 재고 차감
  ├─ group=payment    → 결제
  ├─ group=shipping   → 배송 준비
  └─ group=analytics  → 집계/적재

→ 한 OrderCreated 메시지를 4개 서비스가 "각자" 본다 (group이 다르므로)
→ 각 group은 자기 offset을 독립적으로 관리 (서로의 진행을 모름)
```

> 핵심: **여러 서비스가 같은 메시지를 보게 하려면 각 서비스가 서로 다른 `groupId`를 쓴다.** 같은 groupId로 묶으면 메시지가 한 인스턴스에만 가서 fan-out이 깨진다.

---

## 7. 같은 서비스의 pod가 여러 개면? — 파티션 할당과 rebalance

> [!important] 결론 먼저: race condition이 아니다
> 같은 서비스의 pod들은 **같은 groupId**로 구독한다. Kafka는 그룹 안에서 **파티션을 각 pod에 배타적으로 분배**하므로, **한 파티션은 그룹 내 정확히 한 pod만** 읽는다. 같은 메시지를 두 pod가 동시에 처리하는 일은 없다.

```
order-events (파티션 0,1,2,3)  +  inventory-service group (pod 4개)

  파티션 0 ──► pod A    ┐
  파티션 1 ──► pod B    │ 한 파티션 = 그룹 내 한 pod만 (배타 할당)
  파티션 2 ──► pod C    │ → 같은 메시지 동시 처리 없음 → race 아님
  파티션 3 ──► pod D    ┘
```

### 파티션 수 = 병렬성 상한

```
pod 4, 파티션 4  → 1 pod = 1 파티션 (이상적)
pod 2, 파티션 4  → pod당 2 파티션 담당 (OK)
pod 6, 파티션 4  → pod 2개는 파티션 못 받고 IDLE (놀게 됨!)
```

> **HPA로 pod를 파티션 수 이상 늘려도 소용없다.** 컨슈머 병렬성을 높이려면 파티션을 늘려야 한다(11절). 그래서 파티션 수를 미리 넉넉히 잡는다(늘리긴 쉬워도 줄이긴 어려움).

### Rebalance — pod가 늘거나 죽으면

```
pod 크래시 / HPA scale-out / 새 컨슈머 합류
  → rebalance 발생 → 파티션 재할당
  → 신버전 cooperative rebalancing = 점진적 재분배(전체 stop-the-world 최소화)
```

### 그럼 진짜 주의할 "경쟁"은 따로 있다

메시지 분배는 안전하지만, 세 가지는 별도로 보장해야 한다.

| 우려 | 실제 | 대응 |
|------|------|------|
| 여러 pod가 같은 메시지 동시 처리? | ❌ 안 일어남 (파티션 배타 할당) | 자동 |
| rebalance 시 중복? | ⚠️ 가능 (offset 커밋 전 이동 → 재처리) | **멱등 처리** (메시지 ID 기록) |
| 같은 엔티티 순서 보장? | ⚠️ key 설계에 달림 | **같은 key → 같은 파티션 → 같은 pod** |
| 같은 DB row 동시 수정? | ⚠️ 가능 (다른 메시지를 다른 pod가) | **DB 락 / `@Version`** ← 이게 진짜 race |

> 정리: **Kafka 메시지 분배 자체는 race가 아니다**(파티션이 그룹 내 한 pod에 배타 할당). 다만 ① rebalance 중복 → 멱등성, ② 순서 → key 설계(섹션 4의 `orderId` key), ③ 같은 row 동시 수정 → DB 동시성 제어, 이 셋은 따로 챙긴다.

---

## 8. 메시지는 언제 "complete"되는가 — offset commit의 의미

> [!important] 가장 흔한 오해
> Kafka에서 "메시지 완료(complete)"는 **메시지 삭제가 아니다.** 소비해도 메시지는 retention 기간까지 그대로 남는다. RabbitMQ의 ack(=큐에서 제거)와 근본적으로 다르다.

```
"완료" = 각 consumer group이 그 메시지의 offset을 commit한 것
       = "이 group은 여기까지 처리했다"는 북마크(진행 위치)

offset은 group별로 독립:
  inventory가 offset 100 커밋  ──┐
  payment가   offset  80 커밋   ├─ 서로 무관. 각자의 완료 지점이 다름
  shipping이  offset 100 커밋  ──┘

→ "이 메시지가 complete됐나?"의 답은 group 수만큼 존재한다.
  group마다 따로 완료된다. 전역 "완료" 같은 건 Kafka에 없다.
```

**commit 시점이 전달 보장을 결정한다:**

| 커밋 시점 | 모드 | 위험 | 대응 |
|----------|------|------|------|
| 처리 전 (auto-commit) | at-most-once | 유실 | 거의 안 씀 |
| 처리 후 (manual ack) | at-least-once | 중복 | **멱등 처리 필수** (기본 권장) |
| 트랜잭션 | exactly-once | — | EOS, 오버헤드 큼 |

```java
// 섹션 5의 ack.acknowledge() = "이 group이 이 offset까지 완료"를 브로커에 기록
// 처리 성공 후에 커밋해야 at-least-once (실패 시 재처리됨)
inventoryService.reserve(event.orderId());   // 1. 비즈니스 처리
ack.acknowledge();                            // 2. 그 다음 offset 커밋 = complete
```

---

## 9. 그럼 "전체 워크플로우 완료"는 누가 판단하나

7절의 offset commit은 **그룹별 로컬 완료**일 뿐이다. "모든 서비스가 다 처리해서 주문이 끝났다"는 **전역 완료**는 Kafka가 모른다 — 각 group은 서로의 진행을 모르기 때문이다. 이걸 알려면 비즈니스 레벨 패턴이 필요하다.

### (1) Choreography Saga — 완료 이벤트 체인 (순차)

```
OrderCreated → inventory 처리 → InventoryReserved 발행
            → payment 처리   → PaymentCompleted 발행
            → shipping 처리  → OrderCompleted 발행  ← 마지막 서비스가 "완료" 확정
```
중앙 조정자 없음. **체인의 마지막 단계 서비스**가 완료 상태를 찍는다. 단순하지만 흐름이 코드에 흩어진다.

### (2) Orchestration Saga — 중앙 오케스트레이터 (제어)

```
Orchestrator ──명령──► inventory ──응답──► Orchestrator
             ──명령──► payment   ──응답──► Orchestrator
             ──명령──► shipping  ──응답──► Orchestrator
             → 모든 단계 응답 수신 → "전체 완료" 결정 (실패 시 보상 트랜잭션)
```
**완료 판단 지점 = 오케스트레이터.** 흐름이 한 곳에 모여 추적·보상이 쉽다. (Spring StateMachine, Camunda, Temporal). → [[Saga-Pattern]]

### (3) Aggregator — 병렬 fan-out 완료 집계

6절처럼 한 메시지를 N개 서비스가 **병렬**로 보는 경우, "다 끝났는지"는 완료 이벤트를 모아서 센다.

```java
// 각 서비스는 처리 후 correlationId(=orderId)와 함께 완료 이벤트 발행
// Aggregator가 동일 correlationId의 완료 이벤트 N개를 카운트
@KafkaListener(topics = "task-completed-events", groupId = "completion-aggregator")
public void onTaskCompleted(TaskCompletedEvent e, Acknowledgment ack) {
    long done = completionRepo.markDone(e.correlationId(), e.serviceName()); // 멱등 upsert
    if (done == EXPECTED_SERVICE_COUNT) {              // N개 다 모이면
        orderService.markFullyCompleted(e.correlationId());  // ← 전역 완료 확정
        eventProducer.publish(new OrderCompletedEvent(e.correlationId()));
    }
    ack.acknowledge();
}
```
`correlationId`로 묶고 **카운트가 차는 지점**이 전역 완료. Kafka Streams aggregation이나 DB 카운터로 구현. 멱등(같은 서비스의 중복 완료 이벤트 무시)이 필수다.

---

## 10. 정리 — "complete"의 두 층위

| 층위 | 무엇이 완료되나 | 완료 지점 | 추적 단위 |
|------|----------------|----------|----------|
| **Kafka 레벨** | 한 group의 소비 진행 | `ack.acknowledge()` = offset commit | group별 독립 (그룹 수만큼) |
| **비즈니스 레벨** | 전체 워크플로우 | Saga 마지막 단계 / 오케스트레이터 / Aggregator 카운트 충족 | correlationId(주문 등) 단위 |

> **답**: 메시지의 "complete"는 Kafka에선 *각 consumer group의 offset commit*이고, 이건 그룹마다 따로다. "여러 서비스가 다 처리해서 끝났다"는 전역 완료는 Kafka가 모르므로, **Saga(마지막 단계/오케스트레이터) 또는 correlationId 기반 Aggregator**가 그 판단을 맡는다.

---

## 11. 신뢰성/전달 보장
- **acks=all + min.insync.replicas** → 데이터 손실 방지.
- 기본 **at-least-once** → 컨슈머 **멱등 처리** 필수.
- **Idempotent Producer / Transactions** → exactly-once 시맨틱(EOS) 가능.
- 발행 원자성(DB+Kafka)은 [[Outbox-Pattern]].

## 12. 운영 포인트
- **파티션 수**가 병렬성 상한 → 처리량 설계의 핵심(늘리긴 쉬워도 줄이긴 어려움).
- 컨슈머 lag 모니터링(처리 지연 지표) — 절대값보다 *증가 추세*를 본다.
- 스키마 관리: **Schema Registry**(Avro/Protobuf)로 호환성 유지. → [[Event-Design]]
- KRaft 모드(ZooKeeper 제거)가 최신 표준.

### 12.1 핫 파티션 (Hot Partition)

같은 key는 같은 파티션으로 가므로(순서 보장의 대가), **특정 key에 트래픽이 쏠리면** 그 파티션을 맡은 컨슈머만 과부하된다.

```
예: key=userId인데 한 헤비유저가 전체 이벤트의 30% 발생
  → 그 파티션만 lag 폭증, 나머지 파티션은 한가
대응:
  - key 설계 재검토 (순서가 꼭 필요한 단위인지)
  - 순서 불필요하면 key 없이(라운드로빈) 발행
  - 복합 key (userId + bucket)로 분산
```

### 12.2 메시지 크기 & Claim-Check

Kafka 기본 메시지 한계는 1MB(`max.message.bytes`). 큰 페이로드(이미지·파일)는 직접 싣지 말고 **S3에 저장 후 참조만 전송**(Claim-Check). → [[Event-Design]]

### 12.3 보안

```
전송 암호화:  TLS/SSL (브로커-클라이언트, 브로커 간)
인증:         SASL (SCRAM / OAUTHBEARER / mTLS)
인가:         ACL — 토픽·그룹별 read/write 권한 (principal 단위)
              kafka-acls --add --allow-principal User:svc-order \
                --operation Read --topic order-events --group inventory-service
```
→ 서비스별 최소 권한 원칙. [[Security-Checklist]]·[[Kubernetes]] 시크릿 관리와 연계.

### 12.4 Replay / 재처리
- 로그 보존 덕분에 **offset 되감기**로 재소비(버그 수정 후 재처리, 신규 컨슈머 백필).
  ```bash
  kafka-consumer-groups --reset-offsets --to-earliest \
    --group inventory-service --topic order-events --execute
  ```
- ⚠️ replay 시 **부수효과 재발** → 멱등 소비자([[Inbox-Pattern]]) 전제 필수.

## 13. Kafka vs RabbitMQ
- 대용량·재소비·스트림·로그 보존 → **Kafka**
- 복잡 라우팅·작업 큐·RPC → [[RabbitMQ]]
- **완료(ack)의 의미 차이**: RabbitMQ ack = 큐에서 메시지 제거. Kafka commit = group의 진행 북마크(메시지는 보존). → fan-out·재소비는 Kafka가 자연스럽다.

## 14. 관련
- [[RabbitMQ]] · [[MQ-Concepts]] · [[Event-Sourcing]] · [[CQRS]] · [[Saga-Pattern]] · [[Outbox-Pattern]] · [[MSA]]
