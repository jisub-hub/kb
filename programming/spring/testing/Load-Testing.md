---
tags:
  - testing
  - load-testing
  - performance
  - k6
  - gatling
created: 2026-06-15
---

# 부하/성능 테스트 전략 (Load Testing)

> [!summary] 한 줄 요약
> 부하 테스트의 신뢰도는 **부하 생성기를 측정 대상과 분리**하고 **환경을 프로덕션과 유사하게** 맞추는 데서 나온다. 로컬은 스크립트 개발·상대 비교·프로파일링용, **실사용 부하·용량 산정은 프로덕션 유사(클라우드) 환경**에서 한다.

---

## 1. 테스트 종류
| 종류 | 목적 |
|------|------|
| **Load Test** | 예상 부하에서 SLA(지연/처리량) 충족 확인 |
| **Stress Test** | 한계점(breaking point)까지 밀어붙임 |
| **Spike Test** | 급격한 트래픽 폭증 대응 |
| **Soak/Endurance** | 장시간 부하 → 메모리 누수·자원 고갈 탐지 |
| **Capacity Test** | 목표 부하에 필요한 인프라 규모 산정 |

---

## 2. ⭐ 로컬 부하 테스트의 한계 (왜 절대 수치를 믿으면 안 되나)
```
[로컬] 같은 머신에서 부하생성기 + 앱 + (가끔 DB)까지 자원 공유
  → CPU/메모리/네트워크를 서로 다툼(contention)
  → "앱이 느린 건지 부하생성기가 느린 건지" 구분 불가
  → latency ≈ 0 (비현실적), 단일 머신이라 동시성 생성량 제한
```

> [!danger] 흔한 오해 바로잡기
> "로컬은 **CPU 부하 테스트**에 유용하다" → ❌ 오히려 **가장 부정확**하다.
> 부하생성기가 CPU를 먹으면서 앱과 코어를 다투기 때문. CPU 성능은 **부하 테스트가 아니라 프로파일링**(async-profiler, JFR)으로 단일/소수 동시성에서 본다.

### 로컬이 진짜 유용한 곳
- 부하 테스트 **스크립트 개발/검증**(시나리오가 의도대로 도나)
- **스모크 테스트**(깨지진 않나), 회귀 빠른 확인
- **상대 비교** — *동일 환경에서* A안 vs B안 (절대값 X, 추세만)
- 프로파일링/디버깅

---

## 3. 실사용 부하는 왜 프로덕션 유사(클라우드)인가
핵심은 "클라우드여서"가 아니라 **클라우드라야 아래 조건을 갖추기 쉬워서**다.

| 조건 | 이유 |
|------|------|
| **부하 생성기 분리** | SUT와 자원 경쟁 제거 (가장 중요) |
| **분산 부하 생성** | 단일 머신은 수만 동시성을 못 만든다 → 여러 노드 |
| **프로덕션 유사 인프라** | 동일 DB 사양·[[Connection-Pool]]·LB·네트워크 |
| **실제 네트워크 지연** | 로컬 latency≈0은 비현실적 |
| **운영급 데이터량** | 빈 DB vs 1억 건 → 쿼리 성능 천차만별 |

```
[권장 구성]
 [부하생성 노드 N개] ──네트워크──► [LB] ──► [App (프로덕션 동급)] ──► [DB (프로덕션 동급)]
   (k6/Gatling, 별도 인스턴스)              ↑ 메트릭/추적 수집([[Metrics]],[[Tracing]])
```

> [!warning]
> [[VirtualThreads-vs-WebFlux]] 같은 **동시성 모델 간 처리량 비교는 로컬 금지** — 부하생성기·이벤트루프·앱이 같은 코어를 다퉈 결과가 뒤집힐 수 있다.

---

## 4. 도구
| 도구 | 특징 |
|------|------|
| **k6** (Grafana) | JS 스크립트, CLI, CI 친화, 클라우드(k6 Cloud) 분산 |
| **Gatling** | Scala/Java DSL, 상세 리포트, 고성능 |
| **JMeter** | GUI, 플러그인 풍부, 전통적 |
| **Locust** | Python, 분산 |
| **wrk/wrk2** | 초경량 HTTP 벤치(간단 측정) |

### k6 예시
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp-up
    { duration: '5m', target: 100 },   // 정상 부하 유지
    { duration: '2m', target: 500 },   // spike
    { duration: '2m', target: 0 },     // ramp-down
  ],
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<800'],  // SLA: p95<300ms
    http_req_failed: ['rate<0.01'],                  // 에러율 1% 미만
  },
};

export default function () {
  const res = http.get('https://staging.example.com/api/orders/1');
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(1);
}
```

---

## 5. 측정 지표 & 실험 설계
- **처리량(RPS/TPS)**, **레이턴시 분포(p50/p95/p99)** — 평균이 아닌 **테일** 중시.
- **에러율**, **포화 지표**(CPU/메모리/커넥션풀 대기/DB lock).
- **3축 관측 연계**: [[Metrics]](수치) + [[Tracing]](병목 구간) + [[Logging]](상세).
- 한 번에 **변수 하나만** 바꿔 비교(부하생성 환경 고정).
- **워밍업** 후 측정(JIT 컴파일, 커넥션 풀 예열).
- 병목은 보통 **다운스트림**(DB 커넥션, 외부 API) → [[Connection-Pool]] 함께 모니터링.

## 6. 베스트 프랙티스
- 부하 생성기를 SUT와 **물리적으로 분리**(다른 인스턴스/리전).
- 프로덕션과 **데이터량·인프라 사양**을 맞춘 스테이징.
- CI에 스모크 부하 테스트 통합(회귀 탐지), 대규모는 별도 스케줄.
- 프로덕션 부하 테스트 시 영향 격리(쉐도우 트래픽/카나리).

## 7. 관련
- [[VirtualThreads-vs-WebFlux|가상스레드 vs WebFlux]] · [[Connection-Pool]] · [[Metrics]] · [[Tracing]] · [[MSA]]
