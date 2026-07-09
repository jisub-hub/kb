---
tags: [java, version, java18]
created: 2026-06-15
---

# Java 18 (2022)

> [!summary] 비-LTS. **UTF-8 기본 charset**, 심플 웹서버.

## 핵심 기능
- **UTF-8 기본 문자셋** — 플랫폼 무관 일관성 (`file.encoding=UTF-8`)
- **Simple Web Server** — `jwebserver` (정적 파일 테스트용)
- **Javadoc 코드 스니펫** `@snippet`
- Vector API(2차 incubator), FFM(2차 incubator), 패턴매칭 switch(2차 preview)

## 사용 예시
```bash
jwebserver -p 8000 -d /path/to/static   # 즉석 정적 서버
```
```java
// 이제 OS 로캘과 무관하게 UTF-8 (인코딩 버그 감소)
Files.writeString(Path.of("a.txt"), "한글");
```

## 도입 장단점
- ✅ 인코딩 일관성(특히 한글/멀티바이트 버그 감소), 테스트 편의
- ❌ 기존 코드가 플랫폼 기본 인코딩에 의존했다면 **동작 변화 주의**, 비-LTS

## 관련
- [[Java-17-LTS]] · [[Java-19]]
