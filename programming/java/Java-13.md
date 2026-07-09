---
tags: [java, version, java13]
created: 2026-06-15
---

# Java 13 (2019)

> [!summary] 비-LTS. **텍스트 블록(preview)** 등장.

## 핵심 기능
- **텍스트 블록 (preview)** — `"""..."""` 멀티라인 문자열
- switch 표현식 (2차 preview, `yield` 도입)
- Dynamic CDS, ZGC 메모리 반환 개선

## 사용 예시
```java
// preview (13) — 정식은 15
String json = """
    {
      "name": "%s",
      "age": %d
    }""".formatted(name, age);
```

## 도입 장단점
- ✅ JSON/SQL/HTML 등 멀티라인 가독성 대폭 향상
- ❌ preview 단계, 비-LTS

## 관련
- [[Java-12]] · [[Java-14]] · [[Java-15]]
