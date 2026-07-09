---
tags:
  - spring
  - http-client
  - resilience
  - timeout
  - msa
created: 2026-06-19
---

# Spring HTTP 클라이언트 회복성 (RestClient/WebClient)

> [!summary] 한 줄 요약
> MSA 장애 1순위는 **클라이언트 타임아웃 미설정**이다 — 느린 외부 호출이 무한 대기로 호출 스레드·커넥션을 묶고, 그게 **연쇄 장애(cascading failure)**로 번진다. 타임아웃 3종 + 클라이언트 커넥션 풀 + 멱등 재시도 + 서킷브레이커를 한 세트로 건다.

---

## 1. 왜 — 타임아웃 없는 호출이 시스템을 죽인다

```
A서비스 → (HTTP) → B서비스 (느려짐/행)
  타임아웃 無 → A의 호출 스레드가 무한 대기
  → 톰캣 워커/커넥션 고갈 → A도 응답 불가 → A를 부르는 C도... = 연쇄 붕괴

기본값 함정: RestTemplate·일부 클라이언트는 기본 타임아웃이 "무한"
```
> [!danger] 모든 외부 호출에는 **반드시** 타임아웃을 명시한다. 기본값에 의존하지 말 것.

## 2. 타임아웃 3종 — 의미가 다르다

| 타임아웃 | 의미 | 미설정 시 |
|----------|------|----------|
| **connect** | TCP 연결 수립까지 | 연결 안 되는 호스트에 오래 매달림 |
| **read / response** | 연결 후 응답(바이트) 대기 | 느린 서버에 무한 대기 |
| **전체(전역)** | 요청 시작~완료 총 한도 | 청크가 찔끔찔끔 오면 read만으론 못 막음 |

```java
// RestClient (Spring 6.1+, 동기 · 가상스레드 친화)
RestClient.builder()
    .requestFactory(ClientHttpRequestFactories.get(
        ClientHttpRequestFactorySettings.DEFAULTS
            .withConnectTimeout(Duration.ofSeconds(2))
            .withReadTimeout(Duration.ofSeconds(5))))
    .build();

// WebClient (리액티브) — Reactor Netty
HttpClient http = HttpClient.create()
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
    .responseTimeout(Duration.ofSeconds(5));        // 전체 응답 타임아웃
WebClient.builder().clientConnector(new ReactorClientHttpConnector(http)).build();
```
> read 타임아웃만으론 "느리게 계속 흘리는" 응답을 못 막는다 → **전체 타임아웃**도 함께.

## 3. 클라이언트 커넥션 풀 — DB 풀과 별개 ⚠️

```
HTTP 클라이언트도 커넥션 풀을 쓴다(Apache HttpClient·Reactor Netty)
  maxConnections(per route) 부족 → pending queue 대기 → 지연 폭증·acquire 타임아웃
  과다 → 대상 서버 과부하

핵심: 이건 [[Connection-Pool]](DB)과 완전히 다른 풀.
      대상 서비스별(per-route) 동시 호출 수를 산정해 설정.
```
- Reactor Netty는 기본 글로벌 풀 공유 → 호출처가 많으면 격리(전용 풀)·`maxConnections`·`pendingAcquireTimeout` 조정.

## 4. 재시도 — 멱등한 것만, 안전하게 ⚠️

```
재시도 가능: GET·PUT·DELETE 등 멱등 메서드 (+ 명확히 멱등 보장된 경우)
재시도 위험: POST(주문 생성 등 비멱등) → 중복 실행 → 이중 결제!
  → 비멱등 재시도는 Idempotency-Key 필요 (→ [[REST-API]] 멱등성)

방법:
  - 지수 백오프 + 지터(thundering herd 방지)
  - 재시도 대상 한정: 연결 실패·5xx·타임아웃만. 4xx(클라이언트 잘못)는 재시도 무의미
  - 최대 횟수 + 전체 데드라인(재시도가 타임아웃 예산을 넘기지 않게)
```

## 5. 서킷브레이커·벌크헤드 연동

```
타임아웃/재시도만으론 부족 → 반복 실패하는 대상은 빠르게 차단(서킷 open)
  서킷브레이커: 실패율 임계 초과 시 호출 자체를 막아 빠르게 실패(자원 보호)
  벌크헤드   : 대상별 동시 호출 수 격리 → 한 대상 장애가 전체 스레드 잠식 방지
  → 상세·분산 환경 조정은 [[Resilience4j]] (§8 분산 서킷브레이커)
```
> 순서(권장): 벌크헤드 → 타임아웃 → 서킷브레이커 → 재시도. 재시도를 서킷 안쪽에 두어 open 시 재시도까지 차단.

## 6. RestClient vs WebClient — 무엇을 쓰나

```
RestClient (동기, Spring 6.1+):
  - 코드 단순, 가상스레드(Java 21+)와 결합 시 블로킹 비용 저렴
  - 대부분의 동기 MSA 호출 → 기본 권장 (→ [[VirtualThreads-vs-WebFlux]])
WebClient (리액티브):
  - WebFlux 스택·스트리밍·대규모 동시 호출(논블로킹)
  - 단 전 구간 리액티브여야 이점 (→ [[MVC-vs-WebFlux]])
RestTemplate: 유지보수 모드(신규 비권장) → RestClient로 이전
```

## 7. 체크리스트

```
□ 모든 외부 호출에 connect + read + 전체 타임아웃을 명시했나
□ 타임아웃 합(재시도 포함)이 상위 호출자의 타임아웃보다 짧은가(데드라인 전파)
□ 클라이언트 커넥션 풀을 대상별 동시성에 맞게 산정했나(DB 풀과 별개)
□ 재시도를 멱등 메서드로만 한정했나(POST는 Idempotency-Key)
□ 반복 실패 대상에 서킷브레이커·벌크헤드를 걸었나
□ 호출 구간이 분산추적에 잡히나(→ [[Tracing]])
```

---

## 8. 관련
- [[Resilience4j]] — 서킷브레이커·벌크헤드·재시도(분산 조정 §8)
- [[REST-API]] — 멱등성·Idempotency-Key(재시도 안전성의 근거)
- [[Connection-Pool]] — DB 커넥션 풀(클라이언트 HTTP 풀과 구분)
- [[VirtualThreads-vs-WebFlux]] — RestClient+VT vs WebClient 선택
- [[MVC-vs-WebFlux]] — 동기/리액티브 스택 선택
- [[MSA]] — 서비스 간 호출과 연쇄 장애 맥락
- [[Tracing]] — 외부 호출 구간 추적
