---
tags: [java, version, java23]
created: 2026-06-15
---

# Java 23 (2024)

> [!summary] 비-LTS. Javadoc Markdown, 모듈 임포트 선언(preview). (문자열 템플릿은 보류/철회)

## 핵심 기능
- **Markdown Javadoc** — 주석을 마크다운으로 작성
- **모듈 임포트 선언 (preview)** — `import module M;` 로 모듈 전체 export 임포트
- **패턴의 원시 타입 (preview)** — switch/instanceof에서 `int` 등 직접
- **ZGC 세대별(generational) 기본화**
- Stream Gatherers(2차 preview), 미사용 import 경고 완화
- ⚠️ 문자열 템플릿: 설계 재검토로 **철회**(미정식)

## 사용 예시
```java
// 모듈 임포트 (preview)
import module java.base;     // java.base가 export하는 패키지 일괄 임포트

/// Markdown으로 쓰는 Javadoc
/// - 목록
/// - `code`
```

## 도입 장단점
- ✅ 문서화 편의, 스크립트/학습 시 임포트 간소화
- ❌ 대부분 preview, 비-LTS, 문자열 템플릿 공백

## 관련
- [[Java-22]] · [[Java-24]]
