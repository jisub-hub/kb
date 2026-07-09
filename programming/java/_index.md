---
tags:
  - java
  - moc
  - index
  - versions
created: 2026-06-15
---

# ☕ Java 버전별 변경 MOC (Java 8 이후)

> Java 8 이후 각 메이저 릴리스의 핵심 변화·사용처·장단점. **LTS** 는 별도 표기. (기준: 2026-06 / 세부는 각 릴리스 노트로 검증 권장)

## 버전 목록
- [[Java-09]] — 모듈(JPMS), JShell, 컬렉션 팩토리, Stream 개선
- [[Java-10]] — `var` 지역변수 타입추론
- [[Java-11-LTS]] ⭐ **(LTS)** — 표준 HTTP Client, String 메서드, Java EE 제거
- [[Java-12]] — switch 표현식(preview), Shenandoah
- [[Java-13]] — 텍스트 블록(preview)
- [[Java-14]] — switch 표현식(정식), record/패턴매칭(preview), 친절한 NPE
- [[Java-15]] — 텍스트 블록(정식), sealed(preview), ZGC 정식
- [[Java-16]] — record(정식), instanceof 패턴매칭(정식), Stream.toList()
- [[Java-17-LTS]] ⭐ **(LTS)** — sealed(정식), 패턴매칭 기반 다수
- [[Java-18]] — UTF-8 기본, 심플 웹서버
- [[Java-19]] — 가상 스레드(preview), 구조적 동시성(incubator)
- [[Java-20]] — preview 2차(가상스레드/레코드패턴/스코프드값)
- [[Java-21-LTS]] ⭐ **(LTS)** — 가상 스레드(정식), 패턴매칭 switch(정식), record 패턴(정식), Sequenced Collections
- [[Java-22]] — FFM(정식), 미사용 변수/패턴, 스트림 게더러(preview)
- [[Java-23]] — Markdown Javadoc, 모듈 임포트 선언(preview)
- [[Java-24]] — 다수 preview 정식화, Class-File API, AOT 클래스 로딩
- [[Java-25-LTS]] ⭐ **(LTS)** — 컴팩트 소스/인스턴스 main(정식), 모듈 임포트(정식), 스코프드 값(정식)
- [[Java-26]] — HTTP/3 HttpClient, G1 처리량 개선, GC무관 AOT 캐싱, final 필드 변경 경고, Applet 제거

---

## 🧭 LTS 한눈에
| 버전 | 출시 | 비고 |
|------|------|------|
| **Java 8** | 2014 | 람다/스트림/Optional. 아직도 많이 쓰임 |
| **Java 11** | 2018 | 첫 "현대" LTS. 8 다음 표준 이주 대상 |
| **Java 17** | 2021 | record/sealed/패턴매칭 완성. 광범위 채택 |
| **Java 21** | 2023 | **가상 스레드** — 동시성 패러다임 전환 |
| **Java 25** | 2025 | 최신 LTS. 언어 단순화·정식화 마무리 |

> LTS(Long-Term Support)는 장기 보안/버그 패치를 받는다. **프로덕션은 LTS** 채택이 정석. 비-LTS는 6개월 지원뿐.

## 🔑 큰 흐름
- **8→11**: 모듈·HTTP Client·API 정리, Java EE 분리
- **11→17**: 언어 현대화 — `var`, record, sealed, switch 표현식/패턴, 텍스트 블록
- **17→21**: **가상 스레드**로 동시성 혁신, 패턴 매칭 완성
- **21→25**: FFM·스코프드값·구조적 동시성 정식화, 입문/스크립트 친화(인스턴스 main)

## 언어 핵심 · 설계 원칙
- [[OOP-SOLID]] — 객체지향 SOLID 5원칙 (SRP·OCP·LSP·ISP·DIP)
- [[Java-DI]] — Dependency Injection · IoC Container · Bean 스코프 · 순환의존
- [[Java-AOP]] — Aspect·Pointcut·Advice·프록시 메커니즘·@Transactional 내부 동작
- [[Java-Patterns]] — Lambda·Stream·Optional·Reflection·함수형 패턴

## 관련
- [[VirtualThreads-vs-WebFlux]] · [[MVC-vs-WebFlux]] · [[JPA-Hibernate]]
