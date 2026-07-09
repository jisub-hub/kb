---
tags: [java, version, java17, lts]
created: 2026-06-15
---

# Java 17 (2021) — (LTS) ⭐

> [!summary] **LTS.** record·sealed·패턴매칭·텍스트블록이 모두 정식화된, 광범위하게 채택된 현대 Java 기준.

## 핵심 기능
- **sealed classes/interfaces (정식)** — 계층 제한 → [[DDD]] 모델링·패턴매칭과 시너지
- **패턴 매칭 for switch (preview)**
- 강화된 의사난수 생성기(`RandomGenerator`)
- macOS/Apple Silicon 렌더링, Foreign Function & Memory(incubator)
- ⚠️ RMI Activation 제거, Applet API 폐기, Security Manager 폐기 예고

## 사용처
- 11 → 17 마이그레이션(LTS 갱신). Spring Boot 3.x는 **Java 17이 최소 요구**.
- record + sealed + switch 패턴으로 도메인 모델/대수적 데이터 타입 표현

## 사용 예시
```java
sealed interface Payment permits Card, Cash, Transfer {}
record Card(String no) implements Payment {}
record Cash() implements Payment {}

// preview(17), 정식은 21
String desc = switch (payment) {
    case Card c    -> "카드 " + c.no();
    case Cash ignored -> "현금";
    case Transfer t -> "계좌이체";
};
```

## 도입 장단점
- ✅ 장기 지원 + 현대 언어기능 완성, **Spring Boot 3 / 최신 생태계 표준**
- ❌ 8/11에서 올 때 제거 API(Security Manager 등) 영향 점검 필요

## 관련
- [[Java-16]] · [[Java-21-LTS]] · [[Java-11-LTS]] · [[DDD]]
