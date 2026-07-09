---
tags: [java, version, java26]
created: 2026-06-15
source: openjdk.org/projects/jdk/26, happycoders, infoworld (2026-03)
---

# Java 26 (2026-03-17)

> [!summary] 비-LTS(단기, 6개월 지원). **새 정식 언어 기능은 없음** — 성능 개선·라이브러리 추가·정리가 중심. 총 10개 JEP 중 5개 정식.

> [!info] 출처/기준
> JDK 26 GA: 2026-03-17. 아래는 릴리스 시점 공식 JEP 기준 정리.

## JEP 목록 (10개)
| JEP | 제목 | 상태 |
|-----|------|------|
| **500** | Prepare to Make Final Mean Final (final 필드 변경 경고) | 정식 |
| **504** | Remove the Applet API (Applet API 제거) | 정식 |
| **516** | Ahead-of-Time Object Caching with Any GC | 정식 |
| **517** | HTTP/3 for the HTTP Client API | 정식 |
| **522** | G1 GC: Improve Throughput by Reducing Synchronization | 정식 |
| 524 | PEM Encodings of Cryptographic Objects | 2차 preview |
| 525 | Structured Concurrency | 6차 preview |
| 526 | Lazy Constants | 2차 preview |
| 529 | Vector API | 11차 incubator |
| 530 | Primitive Types in Patterns, instanceof, switch | 4차 preview |

## 핵심 기능 (정식)

### HTTP/3 for HttpClient (JEP 517) ⭐
- 표준 `HttpClient`가 **HTTP/3 (QUIC 기반)** 지원. 불가 시 HTTP/2 자동 폴백.
```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_3)        // HTTP/3 명시 요청
    .build();
HttpResponse<String> res = client.send(request, BodyHandlers.ofString());
```

### G1 GC 동시성 감소 (JEP 522)
- 두 번째 Card Table 도입 → 앱 스레드와 정제(refinement) 스레드가 분리 접근.
- **처리량 5~15% 개선**, 메모리 비용 미미. (설정 변경 불필요, 자동 적용)

### AOT 객체 캐싱 — GC 무관 (JEP 516)
- AOT 캐시를 힙 직접 매핑 대신 **스트리밍**으로 로드 → **모든 GC에서** 시작 속도 개선(Project Leyden 흐름).

### final 필드 변경 경고 (JEP 500)
- 리플렉션으로 `final` 필드를 **변경하면 경고**. 향후 버전에서 예외로 막기 위한 준비 단계.
- 일부 레거시 라이브러리(직렬화/프록시/모킹)가 영향받을 수 있음 → 점검 필요.

### Applet API 제거 (JEP 504)
- `java.applet` 등 완전 제거 (Java 9부터 deprecated).

## 주목할 preview/incubator
- **원시 타입 패턴 매칭(530, 4차 preview)** — switch/instanceof에서 `int` 등 직접 매칭
- **구조적 동시성(525, 6차 preview)** — 가상 스레드 기반 동시 작업 묶음 관리
- **Lazy Constants(526, 2차 preview)** — 지연 초기화 상수
- **Vector API(529)** — SIMD 가속(여전히 incubator)

## 사용처 / 도입 장단점
- ✅ **HTTP/3**로 네트워크 지연·다중화 개선, G1/AOT로 처리량·시작속도, 차세대 암호 대비
- ✅ AI/수치 연산(Vector API), 동시성(구조적 동시성) 준비 기능 검증 가능
- ❌ **비-LTS(6개월 지원)** — 프로덕션은 [[Java-25-LTS]] 권장, 26은 신기능 검증/단명 워크로드용
- ❌ JEP 500 final 경고로 리플렉션 의존 라이브러리 점검 필요, Applet 제거 영향 확인

## 관련
- [[Java-25-LTS]] · [[Java-24]] · [[_index]]
- HTTP/3 ↔ [[REST-API]] · [[Spring-Cloud-Gateway]] / 구조적 동시성 ↔ [[VirtualThreads-vs-WebFlux]]

## 참고
- [JDK 26 (OpenJDK)](https://openjdk.org/projects/jdk/26/)
- [JDK 26: The new features in Java 26 — InfoWorld](https://www.infoworld.com/article/4050993/jdk-26-the-new-features-in-java-26.html)
- [Java 26 Features — HappyCoders](https://www.happycoders.eu/java/java-26-features/)
