---
tags: [java, version, java22]
created: 2026-06-15
---

# Java 22 (2024)

> [!summary] 비-LTS. **FFM(Foreign Function & Memory) 정식화** — 네이티브 연동 표준.

## 핵심 기능
- **Foreign Function & Memory API (정식)** — JNI 대체, 네이티브 라이브러리/메모리 안전 호출
- **미사용 변수·패턴 `_`** — unnamed variables/patterns
- **다중 파일 소스 프로그램 실행** (`java` 런처)
- **Stream Gatherers (preview)** — 커스텀 중간 연산
- super() 이전 문장 허용(preview)

## 사용 예시
```java
// 미사용 변수 _
for (var _ : list) count++;
catch (Exception _) { ... }

// 다중 파일 소스 실행 — 빌드 없이 여러 .java 실행
// $ java Main.java
```

## 도입 장단점
- ✅ FFM으로 네이티브 연동을 안전·고성능으로(JNI 탈피), 문법 잡음 제거(`_`)
- ❌ Stream Gatherers 등은 preview, 비-LTS

## 관련
- [[Java-21-LTS]] · [[Java-23]]
