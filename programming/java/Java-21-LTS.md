---
tags: [java, version, java21, lts]
created: 2026-06-15
---

# Java 21 (2023) — (LTS) ⭐

> [!summary] **LTS.** **가상 스레드 정식화** — 동시성 패러다임을 바꾼 가장 중요한 LTS 중 하나. 패턴 매칭도 완성.

## 핵심 기능
- **가상 스레드 (정식)** — 적은 OS 스레드로 수십만 동시성, 블로킹 코드 유지
- **패턴 매칭 for switch (정식)** + **레코드 패턴 (정식)**
- **Sequenced Collections** — `getFirst()/getLast()/reversed()` 통일 인터페이스
- **Generational ZGC**
- 문자열 템플릿(preview), 구조적 동시성(preview), 스코프드 값(preview)

## 사용처
- **고동시성 I/O 서비스**(웹/게이트웨이)를 리액티브 없이 — Spring Boot 3.2+ `spring.threads.virtual.enabled=true`
- 대수적 데이터 타입 + switch 패턴으로 도메인 분기

## 사용 예시
```java
// 가상 스레드 (정식)
Thread.startVirtualThread(() -> handleRequest());

// 패턴 매칭 switch + record pattern (정식)
String s = switch (shape) {
    case Circle(double r)      -> "원 r=" + r;
    case Rect(double w, double h) when w == h -> "정사각형";
    case Rect r                -> "직사각형";
};

// Sequenced Collections
list.getFirst(); list.getLast(); list.reversed();
```

## 도입 장단점
- ✅ 장기 지원 + **가상 스레드**로 동시성 단순화, 패턴매칭 완성, 컬렉션 API 정리
- ❌ 가상 스레드 함정 주의: `synchronized` 블로킹 시 **pinning**, ThreadLocal 남용 → [[VirtualThreads-vs-WebFlux]]
- ❌ 17 → 21 이주 시 라이브러리 가상스레드 호환 점검

## 관련
- [[Java-17-LTS]] · [[Java-25-LTS]] · [[VirtualThreads-vs-WebFlux]] · [[MVC-vs-WebFlux]]
