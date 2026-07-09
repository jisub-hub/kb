---
tags: [java, version, java24]
created: 2026-06-15
---

# Java 24 (2025)

> [!summary] 비-LTS. 25(LTS) 직전 — 다수 기능 **정식화·성능 기능** 집결.

## 핵심 기능
- **Stream Gatherers (정식)** — 커스텀 중간 연산(`Stream.gather`)
- **Class-File API (정식)** — 바이트코드 생성/분석 표준 API
- **AOT 클래스 로딩 & 링킹** — 시작 속도 개선(Project Leyden 흐름)
- **스코프드 값(정식 근접)**, 구조적 동시성, 키 파생 API
- **양자내성 암호**(ML-KEM/ML-DSA) 추가
- ⚠️ 32비트 x86 제거, Security Manager 영구 비활성

## 사용 예시
```java
// Stream Gatherers — 윈도우/스캔 등 커스텀 연산
var windows = Stream.of(1,2,3,4,5)
    .gather(Gatherers.windowFixed(2))   // [[1,2],[3,4],[5]]
    .toList();
```

## 도입 장단점
- ✅ 스트림 표현력 확장, 시작 성능, 차세대 암호 대비
- ❌ 비-LTS(6개월 지원) — 프로덕션은 **25(LTS)** 권장. 25 직전 검증용

## 관련
- [[Java-23]] · [[Java-25-LTS]]
