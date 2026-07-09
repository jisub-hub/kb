---
tags: [java, version, java15]
created: 2026-06-15
---

# Java 15 (2020)

> [!summary] 비-LTS. **텍스트 블록 정식화**, sealed preview, ZGC 정식.

## 핵심 기능
- **텍스트 블록 (정식)**
- **sealed classes/interfaces (preview)** — 상속 허용 대상 제한
- **ZGC / Shenandoah 정식(production)**
- Hidden classes, EdDSA 서명
- ⚠️ **Nashorn(JS 엔진) 제거**

## 사용 예시
```java
String sql = """
    SELECT id, name FROM member
    WHERE status = 'ACTIVE'""";

// preview (15) — 정식은 17
sealed interface Shape permits Circle, Square {}
```

## 도입 장단점
- ✅ 텍스트 블록 정식, 저지연 GC 정식 → 프로덕션 GC 선택폭↑
- ❌ sealed는 preview, Nashorn 제거 영향 점검, 비-LTS

## 관련
- [[Java-14]] · [[Java-16]] · [[Java-17-LTS]]
