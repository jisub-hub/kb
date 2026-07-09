---
tags:
  - mq
  - rabbitmq
  - amqp
  - messaging
created: 2026-06-15
---

# RabbitMQ

> [!summary] 한 줄 요약
> **AMQP** 기반 전통적 메시지 브로커. **Exchange → Binding → Queue** 라우팅이 매우 유연해 작업 큐, RPC, 복잡한 메시지 분배에 강하다. 메시지는 소비되면 큐에서 사라진다(기본).

---

## 1. 사용처
- **비동기 작업 큐**: 이메일 발송, 이미지 처리 등 백그라운드 작업.
- **서비스 간 비동기 통신** ([[MSA]]).
- **복잡한 라우팅**: 조건별로 여러 컨슈머에 분배.
- **RPC(요청-응답)** 패턴.
- [[Saga-Pattern]] 의 명령/이벤트 전달.

## 2. 핵심 개념
```
Producer ─► Exchange ─(binding/routing key)─► Queue ─► Consumer
```
- **Exchange 타입**
  - `direct`: routing key 정확 일치
  - `topic`: 패턴 매칭(`order.*.created`)
  - `fanout`: 모든 바인딩 큐에 브로드캐스트
  - `headers`: 헤더 기반
- **Queue**: 메시지 저장. **ack** 받으면 삭제.
- **DLX(Dead Letter Exchange)**: 실패/만료 메시지 격리 → [[MQ-Concepts]]

---

## 3. Spring AMQP 설정
```groovy
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual      # 수동 ack (신뢰성)
        retry:
          enabled: true
          max-attempts: 3
```

## 4. Exchange/Queue/Binding 선언
```java
@Configuration
public class RabbitConfig {
    public static final String EXCHANGE = "order.exchange";
    public static final String QUEUE    = "order.created.queue";

    @Bean TopicExchange orderExchange() { return new TopicExchange(EXCHANGE); }

    @Bean Queue orderQueue() {
        return QueueBuilder.durable(QUEUE)
            .withArgument("x-dead-letter-exchange", "order.dlx")   // DLQ 연결
            .build();
    }

    @Bean Binding binding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with("order.created");
    }

    @Bean   // JSON 직렬화
    Jackson2JsonMessageConverter converter() { return new Jackson2JsonMessageConverter(); }
}
```

## 5. Producer
```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public void publish(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfig.EXCHANGE, "order.created", event);   // exchange, routingKey, payload
    }
}
```

## 6. Consumer (수동 ack)
```java
@Component
public class OrderConsumer {

    @RabbitListener(queues = RabbitConfig.QUEUE)
    public void handle(OrderCreatedEvent event, Channel channel,
                       @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            // ... 멱등 처리 (중복 수신 대비) ...
            process(event);
            channel.basicAck(tag, false);                    // 성공 ack
        } catch (Exception e) {
            channel.basicNack(tag, false, false);            // 실패 → DLQ로 (requeue=false)
        }
    }
}
```

## 7. 신뢰성 옵션
- **Publisher Confirms**: 브로커가 메시지 수신 확인.
- **Durable Queue + Persistent Message**: 재시작 후 메시지 유지.
- **Manual Ack**: 처리 완료 후에만 삭제.
- **DLQ + 재시도/백오프**: 실패 메시지 격리·재처리.
- 발행 원자성은 [[Outbox-Pattern]] 으로 보장.

## 8. 운영 심화 (프로덕션) ⭐

> [!warning] RabbitMQ 실장애 대부분은 **prefetch 미설정**과 **HA 큐 타입 오선택**에서 나온다
> 기본기(Exchange/ack)는 쉬워도, 운영 안정성은 아래 4개에서 갈린다.

### 8.1 Prefetch (Consumer QoS) — 한 컨슈머 독식 방지

```
prefetch = "ack 안 한 메시지를 컨슈머당 최대 몇 개까지 미리 보낼지"

기본값(무제한/큰 값)의 함정:
  컨슈머 A가 큐의 메시지를 통째로 선점(버퍼에 쌓아둠)
  → 컨슈머 B/C는 놀고, A만 과부하 → 불균형 + A 죽으면 미ack 전량 재배달

대응: 작게 설정 → 라운드로빈처럼 고르게 분배
  처리 빠름(짧은 작업)  → prefetch 높게(수십~수백): 왕복 오버헤드↓
  처리 느림(긴 작업)    → prefetch 낮게(1~수개): 공정 분배·메모리 보호
  경험칙: prefetch ≈ 동시 처리 가능 수 × (네트워크 RTT 여유분)
```
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 10        # 컨슈머당 미ack 허용치 (기본 250 → 보통 낮춤)
        concurrency: 4      # 컨슈머 스레드 수
```
> [!tip] 긴 작업인데 `prefetch: 250`(기본)이면 한 인스턴스가 250개를 쥐고 나머지가 굶는다. **느린 작업은 거의 항상 prefetch를 낮춰야 한다.**

### 8.2 큐 타입 — Quorum vs Classic Mirrored ⚠️

```
Classic Mirrored Queue (HA 미러링): 🚫 폐지 수순(deprecated, RabbitMQ 3.x 후반~)
  - 마스터-미러 복제, split-brain·동기화 폭주 이슈 다수 → 신규 사용 금지

Quorum Queue: ✅ 현행 권장 HA 큐 (Raft 합의 기반)
  - 과반(quorum) 복제 → 노드 장애 시 자동 리더 선출, 데이터 안전
  - 항상 durable, at-least-once. 멤버 수는 홀수(3,5) 권장
  - 비용: 메시지당 오버헤드↑(Raft 로그), 매우 짧은 TTL/대량 단명 메시지엔 부적합

Classic Queue (비미러, 단일 노드): 복제 불필요한 일시적 작업 큐엔 여전히 OK
```
```java
@Bean
Queue ordersQueue() {
    return QueueBuilder.durable("orders.q")
        .quorum()                                  // Quorum Queue로 선언
        .deliveryLimit(5)                          // 재배달 한도(초과 시 DLX) — quorum 전용
        .build();
}
```
> 결론: **HA가 필요하면 Quorum Queue.** classic mirrored는 신규 도입 금지(폐지됨). 스트림형 대용량 보존·재소비는 큐가 아니라 **Streams** 또는 [[Kafka]].

### 8.3 Lazy Queue — 대량 적체 시 메모리 보호

```
문제: 컨슈머 장애로 메시지가 수백만 건 쌓이면, RAM에 올려둔 큐가 메모리 폭발
      → 브로커가 메모리 워터마크 초과 → 전체 publish 블로킹(클러스터 마비)

Lazy mode: 메시지를 가능한 한 디스크에 보관(메모리 최소 사용)
  - 대량 적체·스파이크 흡수에 안전(메모리 안정)
  - 대신 처리량·지연 약간 손해(디스크 I/O)
  ※ Quorum Queue는 본질적으로 디스크 기반이라 별도 lazy 불필요 —
    신규는 Quorum이 사실상 이 문제를 흡수
```

### 8.4 DLX·TTL·재시도 전략

```
DLX(Dead Letter Exchange)로 보내지는 조건:
  ① basicNack/reject + requeue=false   ② 메시지 TTL 만료
  ③ 큐 길이 초과(max-length)            ④ quorum delivery-limit 초과

재시도 안티패턴 ⚠️:
  requeue=true로 즉시 재큐 → 같은 메시지가 즉시 다시 실패 → 무한 루프(hot loop)
  → CPU 100%·로그 폭주

올바른 백오프:
  실패 → DLX → "지연 큐"(TTL 부여) → TTL 만료 후 원본으로 되돌림
  또는 rabbitmq-delayed-message-exchange 플러그인으로 지연 재시도
  재시도 횟수는 헤더(x-death count)로 추적 → N회 초과 시 parking lot 큐로 격리
```
```yaml
# Spring Retry 기반 (간단한 경우) — 컨슈머 내 재시도 + 소진 시 DLQ
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
          multiplier: 2.0        # 1s → 2s → 4s 지수 백오프
        default-requeue-rejected: false   # 소진 시 requeue 말고 DLQ로
```

### 8.5 신뢰성 조합 — publisher confirm + consumer ack

```
양끝을 모두 잠가야 end-to-end at-least-once:
  Producer 측: Publisher Confirms — 브로커가 "받았다" 확인 못 하면 재발행
               (+ Mandatory: 라우팅 실패 시 return 콜백)
  Broker 측 : Durable Queue + Persistent Message (재시작 생존)
  Consumer 측: Manual Ack — 처리 "완료 후" ack (예외 시 nack→DLX)

⚠️ at-least-once = 중복 수신 가능 → 컨슈머는 반드시 멱등(idempotent)
   (→ [[REST-API]] 멱등성·[[Inbox-Pattern]] 식 중복 제거)
발행 원자성(DB 트랜잭션과 메시지 발행 일치)은 [[Outbox-Pattern]].
```

## 9. RabbitMQ vs Kafka
- 복잡한 라우팅·작업 분배·RPC → **RabbitMQ**
- 대용량 이벤트 스트림·재소비·로그 보존 → [[Kafka]]

## 10. 관련
- [[Kafka]] · [[MQ-Concepts]] · [[Saga-Pattern]] · [[Outbox-Pattern]] · [[MSA]]
- [[Resilience4j]] (재시도·백오프 일반) · [[Tracing]] (메시징 경계 추적)
