---
tags:
  - pattern
  - design-pattern
  - structural
  - adapter
  - hexagonal
created: 2026-06-17
---

# Adapter Pattern (어댑터 패턴)

> [!summary] 한 줄 요약
> **호환되지 않는 두 인터페이스를 중간에서 변환**해 함께 동작하게 하는 구조(structural) 패턴. "내가 원하는 인터페이스(Target)"와 "이미 존재하는 구현(Adaptee)" 사이에 변환기를 끼운다. Spring·[[Hexagonal-Architecture]]의 "Ports & Adapters"가 이 패턴을 아키텍처 수준으로 확장한 것이다.

---

## 1. 개념 — 콘센트 변환 어댑터

110V 기기를 220V 콘센트에 꽂으려면 변환 어댑터가 필요하다. 코드도 같다 — **내 코드가 기대하는 인터페이스**와 **외부/레거시 구현**의 모양이 안 맞을 때, 둘을 잇는 변환 계층을 둔다.

```
[Client] ──기대──► [Target 인터페이스]
                         ▲
                         │ 구현(변환)
                    [Adapter] ──위임──► [Adaptee (기존 구현)]
```

- **Target**: 클라이언트가 호출하고 싶은 인터페이스 (내 도메인 언어).
- **Adaptee**: 이미 존재하지만 인터페이스가 다른 구현 (외부 SDK·레거시·서드파티).
- **Adapter**: Target을 구현하면서, 내부적으로 Adaptee를 호출해 변환.

> 핵심: **기존 코드(Adaptee)를 고치지 않고** 새 인터페이스에 맞춘다. 외부 라이브러리·레거시를 건드릴 수 없을 때 특히 유효.

---

## 2. 두 가지 형태

| 형태 | 방식 | 특징 |
|------|------|------|
| **Object Adapter** (권장) | Adaptee를 **합성(composition)** 으로 보유 | 유연, 여러 Adaptee 가능, Java에서 표준 |
| **Class Adapter** | Adaptee를 **상속** | 단일 상속 제약(Java), 강결합 |

Java는 다중 상속이 없어 **Object Adapter(합성)** 가 사실상 표준이다.

---

## 3. 구현 예시 (Java)

외부 결제 SDK(`LegacyPgClient`)를 우리 도메인 인터페이스(`PaymentGateway`)에 맞추는 경우:

```java
// Target — 우리 도메인이 원하는 인터페이스
public interface PaymentGateway {
    PaymentResult pay(Money amount, Card card);
}

// Adaptee — 외부 SDK (모양이 다름, 우리가 못 고침)
public class LegacyPgClient {
    public PgResponse requestPayment(int won, String cardNo, String expiry) { ... }
}

// Adapter — Target 구현 + Adaptee 호출 변환
@Component
public class LegacyPgAdapter implements PaymentGateway {
    private final LegacyPgClient client;   // 합성 (Object Adapter)

    public LegacyPgAdapter(LegacyPgClient client) { this.client = client; }

    @Override
    public PaymentResult pay(Money amount, Card card) {
        // 도메인 타입 → SDK 타입으로 변환
        PgResponse res = client.requestPayment(
            amount.toWon(), card.number(), card.expiry());
        // SDK 응답 → 도메인 타입으로 변환
        return res.isOk()
            ? PaymentResult.success(res.getTxId())
            : PaymentResult.fail(res.getErrorCode());
    }
}
```

> 도메인 코드는 `PaymentGateway`만 알면 된다. 외부 SDK가 바뀌면 **Adapter만 교체**하면 된다.

---

## 4. Hexagonal Architecture와의 관계 ⭐

[[Hexagonal-Architecture]](Ports & Adapters)는 어댑터 패턴을 **아키텍처 전체 원칙**으로 확장한 것이다.

```
        ┌─────────────────────────────┐
        │      도메인 / 애플리케이션      │
        │   (Port = 인터페이스 정의)     │
        └─────────────────────────────┘
          ▲ inbound port      ▼ outbound port
   [Adapter]                     [Adapter]
   REST Controller          JPA Repository 구현
   Kafka Consumer           외부 API Client
   (driving)                (driven)
```

- **Port** = 도메인이 정의한 인터페이스(Target). "무엇이 필요한가"만 선언.
- **Adapter** = 그 Port를 기술(REST·JPA·Kafka·외부 SDK)로 구현.
- 도메인은 기술을 모른 채 Port에만 의존 → **기술 교체가 Adapter 교체로 국한**.

| 방향 | Adapter 예시 | 역할 |
|------|-------------|------|
| **Inbound(driving)** | REST Controller, Kafka Listener, gRPC | 외부 요청 → 도메인 호출 |
| **Outbound(driven)** | JPA Repository, Redis, 외부 API Client | 도메인 → 외부 기술 호출 |

> [[Repository-Pattern]]도 본질은 outbound adapter다 — 도메인의 저장 Port를 JPA/MyBatis로 어댑팅.

---

## 5. Spring에서 만나는 어댑터들

```
HandlerAdapter        — DispatcherServlet이 다양한 핸들러(@Controller, 함수형 등)를
                        동일 방식으로 호출하도록 변환 (Spring MVC 내부)
HandlerInterceptorAdapter (구버전) — 인터페이스 기본 구현 제공
WebMvcConfigurer       — 설정 변환
외부 연동 어댑터        — PG·SMS·알림·LLM API 등을 도메인 Port로 감쌈 (가장 흔한 실무 용례)
```

실무에서 가장 가치 있는 사용: **외부 시스템(결제·알림·LLM·레거시)을 도메인 Port로 감싸 격리**. 외부 의존성 변경·교체·테스트(목 주입)가 쉬워진다.

---

## 6. 유사 구조 패턴 비교

| 패턴 | 목적 | 인터페이스 |
|------|------|-----------|
| **Adapter** | 기존 인터페이스를 **다른 모양**으로 변환 | 바꿈 |
| **Facade** | 복잡한 서브시스템을 **단순한 창구**로 | 새로 단순화 |
| **Proxy** | 같은 인터페이스로 **접근 제어/지연/캐싱** | 동일 |
| **Decorator** | 같은 인터페이스로 **기능 추가** | 동일 |
| **Bridge** | 추상-구현을 **독립적으로 확장** | 분리(설계 시점) |

> 헷갈림 포인트: Adapter는 "이미 있는 것을 끼워맞춤(사후)", Bridge는 "처음부터 분리 설계(사전)". Decorator/Proxy는 인터페이스를 **유지**, Adapter는 **변환**.

---

## 7. 언제 쓰나 / 주의

```
✅ 외부 SDK·레거시·서드파티의 인터페이스가 내 도메인과 안 맞을 때
✅ 외부 의존성을 도메인에서 격리하고 싶을 때 (Hexagonal)
✅ 같은 역할의 여러 공급자(PG사 A/B/C)를 동일 Port로 통일할 때

⚠️ 주의
- 어댑터 남발 → 변환 계층만 두껍고 의미 없는 위임(과설계)
- 변환 로직이 복잡해지면 매핑 책임을 별도 Mapper로 분리
- "그냥 인터페이스 하나 더"가 아니라, 변환이 실제로 필요한지 판단
```

---

## 8. 정리

> Adapter = **모양이 안 맞는 둘을 잇는 변환기**. 객체 수준에선 외부 SDK를 도메인 인터페이스에 맞추는 클래스이고, 아키텍처 수준에선 [[Hexagonal-Architecture]]의 Port를 기술로 구현하는 계층이다. 핵심 가치는 **외부/기술 의존성을 도메인에서 격리**해 교체·테스트를 쉽게 만드는 것.

---

## 관련
- [[Hexagonal-Architecture]] — Ports & Adapters (어댑터 패턴의 아키텍처 확장)
- [[Repository-Pattern]] — 영속성 outbound adapter
- [[Mediator-Pattern]] — 또 다른 GoF 패턴 (행위)
- [[Spring-Core]] — DI로 Adapter 주입·교체
- [[DDD]] — 도메인 격리, anti-corruption layer(어댑터의 도메인 버전)
- [[MSA]] — 외부 서비스 연동 어댑터
