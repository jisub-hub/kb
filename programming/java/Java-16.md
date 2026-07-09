---
tags: [java, version, java16]
created: 2026-06-15
---

# Java 16 (2021)

> [!summary] 비-LTS. **record·instanceof 패턴매칭 정식화**.

## 핵심 기능
- **record (정식)** — 불변 데이터 운반 클래스 (DTO/VO에 최적) → [[DTO]]
- **instanceof 패턴 매칭 (정식)**
- **`Stream.toList()`** (간결 수집)
- Vector API(incubator), FFI(incubator)
- **JDK 내부 강한 캡슐화 기본화** (리플렉션 우회 차단 강화)
- OpenJDK Git/GitHub 이전

## 사용 예시
```java
record Member(Long id, String email) {}          // 정식
Member m = new Member(1L, "a@b.com");
m.email();                                        // 자동 접근자

if (obj instanceof Member mem && mem.id() > 0) { use(mem); }
List<String> names = stream.map(...).toList();   // 불변 리스트
```

## 도입 장단점
- ✅ record로 보일러플레이트 격감(DTO/VO), 패턴매칭 정식
- ❌ 내부 API 강한 캡슐화로 일부 레거시/라이브러리 깨질 수 있음, 비-LTS

## 관련
- [[Java-15]] · [[Java-17-LTS]] · [[DTO]]
