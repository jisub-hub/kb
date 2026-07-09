---
tags: [java, version, java19]
created: 2026-06-15
---

# Java 19 (2022)

> [!summary] 비-LTS. **가상 스레드(preview)** 첫 등장 — 동시성 전환의 시작.

## 핵심 기능
- **가상 스레드 Virtual Threads (preview)** — 경량 스레드, 블로킹 코드로 고동시성
- **구조적 동시성 Structured Concurrency (incubator)**
- **레코드 패턴 Record Patterns (preview)** — 구조 분해
- 패턴매칭 switch(3차 preview), FFM(preview), 스코프드 값 준비

## 사용 예시
```java
// preview (19) — 정식은 21
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> { Thread.sleep(1000); return i; }));
}   // 수만 개도 OS 스레드 고갈 없이

// record pattern (preview)
if (obj instanceof Point(int x, int y)) { use(x, y); }
```

## 도입 장단점
- ✅ I/O 바운드 고동시성을 **블로킹 스타일 그대로** (리액티브 복잡도 회피) → [[VirtualThreads-vs-WebFlux]]
- ❌ 가상스레드/레코드패턴 모두 preview, 비-LTS — 평가/실험용

## 관련
- [[Java-18]] · [[Java-20]] · [[Java-21-LTS]] · [[VirtualThreads-vs-WebFlux]]
