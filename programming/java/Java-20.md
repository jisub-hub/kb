---
tags: [java, version, java20]
created: 2026-06-15
---

# Java 20 (2023)

> [!summary] 비-LTS. 주요 기능들의 **추가 preview/incubator 라운드** (안정화 단계).

## 핵심 기능
- 가상 스레드 (2차 preview)
- 레코드 패턴 (2차 preview)
- 패턴매칭 switch (4차 preview)
- 구조적 동시성 (2차 incubator)
- **스코프드 값 Scoped Values (incubator)** — ThreadLocal 대안
- FFM (preview)

## 사용 예시
```java
// 스코프드 값 (incubator) — 가상 스레드 친화 컨텍스트 전파
final static ScopedValue<User> CURRENT = ScopedValue.newInstance();
ScopedValue.where(CURRENT, user).run(() -> handle());   // 불변·범위 한정
```

## 도입 장단점
- ✅ 21에서 정식화될 기능을 미리 검증 가능
- ❌ 대부분 preview/incubator, 비-LTS → **실서비스 도입 X**, 21을 기다리는 단계

## 관련
- [[Java-19]] · [[Java-21-LTS]]
