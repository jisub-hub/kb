---
tags: [java, version, java25, lts]
created: 2026-06-15
---

# Java 25 (2025) — (LTS) ⭐

> [!summary] **최신 LTS.** 언어 단순화(입문/스크립트 친화)와 동시성·네이티브 기능의 정식화를 마무리.

## 핵심 기능
- **컴팩트 소스 파일 & 인스턴스 `main` 메서드 (정식)** — `class` 없이 간단 실행, 입문/스크립트 친화
- **모듈 임포트 선언 (정식)** — `import module M;`
- **유연한 생성자 본문 (정식)** — `super()`/`this()` 호출 전 문장 허용
- **스코프드 값 Scoped Values (정식)** — ThreadLocal 대안, 가상 스레드 친화
- **구조적 동시성**(계속 안정화), **Stable Values (preview)**
- **PEM 인코딩 API**, 세대별 Shenandoah, AOT 개선

## 사용처
- 21 → 25 LTS 갱신. 가상 스레드 + 스코프드 값으로 **현대 동시성 스택** 완성
- 인스턴스 main으로 교육/PoC/스크립트 진입장벽↓

## 사용 예시
```java
// 컴팩트 소스 + 인스턴스 main (정식) — 클래스 선언 불필요
void main() {
    IO.println("Hello");        // 간결 입출력
}

// 스코프드 값 (정식)
final static ScopedValue<TraceId> TRACE = ScopedValue.newInstance();
ScopedValue.where(TRACE, traceId).run(this::handle);
```

## 도입 장단점
- ✅ **최신 장기 지원**, 동시성/네이티브/암호 정식화 총정리, 진입장벽 완화
- ✅ 21의 가상 스레드 기반 위에 스코프드 값·구조적 동시성으로 컨텍스트 전파/취소를 안전하게
- ❌ 생태계(프레임워크·빌드·라이브러리) 25 지원 성숙도 확인 후 채택. 비-LTS에서 올렸다면 누적 변경 점검

## 관련
- [[Java-21-LTS]] · [[Java-24]] · [[VirtualThreads-vs-WebFlux]]
