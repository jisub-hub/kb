---
tags: [java, version, java14]
created: 2026-06-15
---

# Java 14 (2020)

> [!summary] 비-LTS. **switch 표현식 정식화**, record·패턴매칭 preview.

## 핵심 기능
- **switch 표현식 (정식)**
- **record (preview)** — 불변 데이터 클래스
- **instanceof 패턴 매칭 (preview)**
- **친절한 NullPointerException** (어떤 변수가 null인지 메시지로)
- JFR Event Streaming, packaging tool(jpackage, incubator)

## 사용 예시
```java
// 정식
String type = switch (obj) { case 1 -> "one"; default -> "other"; };

// preview
record Point(int x, int y) {}                    // 14 preview
if (obj instanceof String s) { use(s); }         // 14 preview, 캐스팅 생략
```

## 도입 장단점
- ✅ switch 표현식 정식 사용 가능, NPE 디버깅 크게 개선
- ❌ record/패턴매칭은 아직 preview, 비-LTS

## 관련
- [[Java-13]] · [[Java-15]] · [[Java-16]]
