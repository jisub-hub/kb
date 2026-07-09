---
tags: [java, version, java9]
created: 2026-06-15
---

# Java 9 (2017)

> [!summary] 비-LTS. Java 8 이후 첫 메이저. **모듈 시스템(JPMS)** 이 최대 변화.

## 핵심 기능
- **JPMS (Project Jigsaw, `module-info.java`)** — JDK/앱을 모듈로 분리, 강한 캡슐화
- **JShell** — REPL (대화형 실행)
- **컬렉션 팩토리 메서드** — `List.of()`, `Map.of()` (불변)
- **Stream 개선** — `takeWhile`, `dropWhile`, `ofNullable`, `iterate(seed, pred, next)`
- **`Optional.ifPresentOrElse` / `stream()`**
- **private interface methods**, Process API 개선, HTTP/2 Client(incubator)

## 사용처
- 대형 모듈러 애플리케이션의 의존성 경계 명확화
- 불변 컬렉션 간결 생성, 스트림 표현력 향상

## 사용 예시
```java
List<String> names = List.of("a", "b", "c");          // 불변
Stream.of(1,2,3,4,5).takeWhile(n -> n < 4)            // [1,2,3]
      .forEach(System.out::println);
opt.ifPresentOrElse(System.out::println, () -> log.warn("empty"));
```

## 도입 장단점
- ✅ 모듈 캡슐화, API 간결화, 스트림/Optional 실용성↑
- ❌ **모듈 시스템은 학습/마이그레이션 비용 큼** — 라이브러리 호환 문제로 실제 채택은 더딤. 대부분 모듈 없이 새 API만 활용

## 관련
- [[Java-10]] · [[Java-11-LTS]]
