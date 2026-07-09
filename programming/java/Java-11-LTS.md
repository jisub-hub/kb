---
tags: [java, version, java11, lts]
created: 2026-06-15
---

# Java 11 (2018) — (LTS) ⭐

> [!summary] **LTS.** Java 8 다음의 표준 이주 대상. 현대 Java의 사실상 기준선.

## 핵심 기능
- **표준 HTTP Client** (`java.net.http`, HTTP/2·비동기·WebSocket) — 정식화
- **String 메서드** — `strip()`, `isBlank()`, `repeat(n)`, `lines()`
- **`Files.readString` / `writeString`**
- **람다 파라미터에 `var`** (`(var x, var y) -> ...`)
- **단일 파일 소스 실행** — `java Hello.java` (컴파일 없이)
- ZGC(실험), Epsilon GC, Flight Recorder 오픈소스화
- ⚠️ **Java EE / CORBA 모듈 제거** (JAXB, JAX-WS 등 → 외부 의존성으로)

## 사용처
- 8 → 11 마이그레이션(보안/지원 종료 대응), 외부 HTTP 클라이언트 라이브러리 대체
- 컨테이너 친화(인지 개선), 스크립트성 단일 파일 실행

## 사용 예시
```java
HttpClient client = HttpClient.newHttpClient();
HttpResponse<String> res = client.send(
    HttpRequest.newBuilder(URI.create("https://api.example.com")).build(),
    HttpResponse.BodyHandlers.ofString());

"  hi  ".strip();      // "hi"
"-".repeat(10);        // "----------"
Files.writeString(Path.of("a.txt"), "hello");
```

## 도입 장단점
- ✅ 장기 지원, 내장 HTTP Client, API 정리, 컨테이너/성능 개선
- ❌ **Java EE 모듈 제거**로 일부 앱은 의존성 추가 필요. 모듈 시스템 호환 점검

## 관련
- [[Java-10]] · [[Java-12]] · [[Java-17-LTS]]
