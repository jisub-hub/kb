---
tags: [java, version, java10]
created: 2026-06-15
---

# Java 10 (2018)

> [!summary] 비-LTS. **`var` 지역 변수 타입 추론**이 대표 기능.

## 핵심 기능
- **`var`** — 지역 변수 타입 추론 (선언부 한정)
- **G1 GC 병렬 Full GC** (지연 개선)
- `Optional.orElseThrow()` (인자 없는 버전)
- 불변 컬렉션 `copyOf`, `Collectors.toUnmodifiable*`
- Application Class-Data Sharing

## 사용처
- 긴 제네릭 타입 선언 간소화, 가독성 향상

## 사용 예시
```java
var list = new ArrayList<Map<String, List<Integer>>>();   // 타입 추론
var entry = map.entrySet().iterator().next();
// 단, 필드/매개변수/반환타입에는 못 씀 (지역 변수만)
```

## 도입 장단점
- ✅ 보일러플레이트 감소, 가독성(우변이 명확할 때)
- ❌ 우변이 불명확하면 오히려 가독성 저하 → **타입이 자명할 때만** 사용 권장

## 관련
- [[Java-09]] · [[Java-11-LTS]]
