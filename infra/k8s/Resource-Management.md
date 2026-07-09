---
tags:
  - kubernetes
  - resource-management
  - jvm
  - performance
  - sre
created: 2026-06-19
---

# K8s 리소스 관리 & JVM 컨테이너 인식

> [!summary] 한 줄 요약
> K8s 백엔드 장애 단골 3종은 **CPU throttling**(limit=CFS quota로 p99 폭증), **OOMKill**(메모리 limit 초과 즉시 종료), **JVM의 컨테이너 오인식**(힙/스레드를 노드 전체 기준으로 잡음). requests/limits를 "그냥 채우는 값"이 아니라 **QoS·스케줄링·런타임 동작**으로 이해해야 한다.
> (requests/limits 기본은 [[Pod]] §4, JVM 힙 산정은 [[JVM-Tuning]] — 여기선 그 둘을 잇는 함정에 집중.)

---

## 1. requests vs limits — 역할이 다르다

```
requests: 스케줄러가 "이만큼은 보장"하고 노드에 배치하는 기준 (예약)
limits:   런타임이 강제하는 상한 (초과 시 CPU=throttle, Memory=OOMKill)

requests < 실사용 → 노드 오버커밋 → 이웃 파드와 경합·eviction
requests = limits → 예측 가능·안정(단 비용↑, 빈 자원 못 빌림)
```

## 2. QoS Class & Eviction 순서 ⚠️

노드 메모리 압박 시 **무엇부터 죽이는지**가 QoS class로 결정된다.

| QoS Class | 조건 | eviction 우선순위 |
|-----------|------|------------------|
| **Guaranteed** | 모든 컨테이너 requests=limits(cpu·mem 둘 다) | 가장 나중(가장 안전) |
| **Burstable** | requests < limits 또는 일부만 설정 | 중간 |
| **BestEffort** | requests/limits 전혀 없음 | **가장 먼저 죽음** |

```
노드 메모리 부족 → kubelet이 BestEffort → Burstable(requests 초과분 큰 순) → Guaranteed 순으로 evict
교훈: 중요한 워크로드는 Guaranteed로(특히 메모리 request=limit) → eviction 표적에서 제외
```

## 3. CPU Throttling — limit이 오히려 지연을 키운다 ⚠️

> [!danger] CPU limit = CFS quota → 멀티스레드 앱이 quota를 조기 소진하면 **강제 대기(throttle)** → p99 지연 폭증
> CPU는 메모리와 달리 "압축 가능 자원"이라 초과 시 죽이지 않고 **조인다**.

```
limit=1(=100ms/100ms quota)인 앱이 8 스레드로 동시 실행
  → 12.5ms만에 100ms 어치 CPU 소진 → 남은 87.5ms는 throttle(정지)
  → 평균 CPU는 낮은데 p99만 튀는 "유령 지연"의 전형

진단: container_cpu_cfs_throttled_periods_total / ..._periods_total (throttle 비율)
       Grafana에서 이 비율이 높으면 limit이 원인
```
- **권장 논쟁**: 다수 SRE 관행은 *CPU limit을 아예 안 걸거나 넉넉히* + requests로 공정 분배(throttle 회피). 단 멀티테넌트·시끄러운 이웃 차단이 필요하면 limit 유지.
- HPA가 CPU 기준이면 throttle과 상호작용 주의 → [[HPA]].

## 4. OOMKill — 메모리는 압축 불가

```
컨테이너 메모리 working set > limit  → 커널 OOM Killer가 PID 1 종료(Exit 137)
  - cgroup 단위로 동작. RSS+page cache 일부 포함(working set)
  - JVM은 힙 외 메타스페이스·스레드 스택·Direct Buffer·코드캐시도 먹음
    → "힙은 limit보다 작은데 OOMKill" = 비힙 영역 누락 계산 ([[JVM-Tuning]])
권장: 메모리는 request = limit (Guaranteed) + limit는 힙 + 비힙 + 여유로 산정
```

## 5. JVM 컨테이너 인식 — CPU 수 오인식이 스레드를 망친다

```
JVM은 cgroup을 읽어 "가용 CPU/메모리"를 추정(Java 11+ UseContainerSupport 기본 on)
  메모리: MaxRAMPercentage로 힙 자동 산정 (→ [[JVM-Tuning]] 상세)
  CPU   : availableProcessors()가 cgroup CPU quota 기반으로 계산

함정: 이 값이 GC 스레드·ForkJoinPool.commonPool·커넥션 풀 기본값을 좌우
  CPU limit을 0.5로 주면 availableProcessors()=1 → 병렬 GC/스트림 병렬도 붕괴
  반대로 limit 미설정 시 노드의 64코어로 인식 → GC 스레드 수십 개 과생성
```
- 명시 제어: `-XX:ActiveProcessorCount=N`으로 JVM이 인식할 CPU 수 고정(오토스케일 환경에서 안정).
- 가상스레드·ForkJoinPool 병렬도도 이 값에 묶임 → [[VirtualThreads-vs-WebFlux]].

## 6. 권장 기본값 (출발점)

```
메모리: request = limit (Guaranteed, OOMKill 예측 가능)
        limit = JVM 힙(MaxRAMPercentage 75%) + 비힙 + 여유 ≈ 힙 / 0.75 + α
CPU   : request = 평소 사용량(공정 분배 보장)
        limit = 미설정 또는 넉넉히(throttle 회피) — 멀티테넌트면 유지
관측  : cfs_throttled 비율·OOMKill 카운트·working set을 항상 대시보드에 ([[Metrics]])
```

## 7. 관련
- [[Pod]] — requests/limits·probe·lifecycle 기초
- [[JVM-Tuning]] — 힙(MaxRAMPercentage)·메타스페이스·Direct Buffer·GC 상세
- [[HPA]] — CPU 기준 오토스케일과 throttle 상호작용
- [[Connection-Pool]] — CPU 인식 오류가 풀·스레드 기본값에 주는 영향
- [[VirtualThreads-vs-WebFlux]] — ForkJoinPool 병렬도가 ActiveProcessorCount에 종속
- [[Metrics]] — throttle·OOMKill·working set 모니터링
