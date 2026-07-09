---
tags: [java, version, java12]
created: 2026-06-15
---

# Java 12 (2019)

> [!summary] 비-LTS. **switch 표현식(preview)** 등장.

## 핵심 기능
- **switch 표현식 (preview)** — 화살표 문법, 값 반환
- **Shenandoah GC** (저지연, 실험)
- `Collectors.teeing` — 두 컬렉터 결합
- `String.indent`, `String.transform`
- CDS 기본 활성화

## 사용 예시
```java
// preview (12) — 정식은 14
int days = switch (month) {
    case JAN, MAR, MAY -> 31;
    case FEB -> 28;
    default -> 30;
};
double avg = nums.stream().collect(
    Collectors.teeing(summingDouble(d->d), counting(), (s,c) -> s/c));
```

## 도입 장단점
- ✅ switch 표현식으로 분기 코드 간결·안전(fall-through 제거)
- ❌ 핵심 기능 다수가 **preview** → 프로덕션엔 부적합, 비-LTS

## 관련
- [[Java-11-LTS]] · [[Java-13]] · [[Java-14]]
