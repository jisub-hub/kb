---
tags:
  - deploy
  - deployment-strategy
  - blue-green
  - canary
  - rolling
  - kubernetes
created: 2026-06-17
---

# 배포 전략 (Blue-Green / Canary / Rolling)

> [!summary] 한 줄 요약
> 무중단 배포의 목표는 **사용자가 배포를 눈치채지 못하게** 새 버전으로 트래픽을 옮기는 것. Rolling은 K8s 기본, Blue-Green은 즉시 전환·즉시 롤백, Canary는 일부 트래픽으로 점진 검증. 어떤 전략이든 핵심은 **readiness probe로 준비 확인 + 메트릭으로 판정 + 빠른 롤백 경로** 확보다.

---

## 1. 무중단 배포 목표

```
[무중단 배포가 보장해야 할 것]
  ① 가용성    — 배포 중에도 요청이 끊기지 않음 (다운타임 0)
  ② 안전성    — 문제 버전이 전체 트래픽을 받기 전에 차단
  ③ 롤백 속도 — 이상 감지 시 즉시 이전 버전으로 복귀
  ④ 데이터 호환 — 구/신 버전이 같은 DB 스키마에서 동시 동작 가능

[전제 조건]
  - 앱이 stateless (세션은 외부 저장소 — Redis 등)
  - graceful shutdown (SIGTERM 수신 → in-flight 요청 처리 후 종료)
  - readiness probe (준비 안 된 인스턴스에 트래픽 차단)
  - 하위 호환 가능한 DB 마이그레이션 (expand-contract)
```

---

## 2. 전략 비교

| 전략 | 다운타임 | 롤백 속도 | 추가 리소스 | 위험도 | 핵심 |
|------|---------|----------|------------|--------|------|
| **Recreate** | 있음 (전체 중단) | 느림 (재배포) | 없음 (×1) | 높음 | 기존 종료 후 신규 기동 |
| **Rolling** | 없음 | 보통 (역롤링) | 적음 (+maxSurge) | 중간 | 파드 단위 점진 교체 |
| **Blue-Green** | 없음 | 즉시 (트래픽 스위치) | 많음 (×2) | 낮음 | 두 환경 동시 유지 후 전환 |
| **Canary** | 없음 | 즉시 (트래픽 회수) | 보통 (+카나리) | 가장 낮음 | 일부 트래픽으로 점진 검증 |
| **Shadow** | 없음 | N/A (응답 무시) | 많음 (복제 트래픽) | 매우 낮음 | 실트래픽 복제로 검증만 |

```
[트래픽 관점 비교]
  Recreate   : v1 ────╳         ┌──── v2     (사이에 공백 = 다운타임)
  Rolling    : v1▓▓░░░░░░  →  ░░░░░░▓▓ v2   (점차 v2 비율 증가)
  Blue-Green : v1(100%) ──[switch]──→ v2(100%)  (순간 전환)
  Canary     : v1(95%) + v2(5%) → v2(50%) → v2(100%)
  Shadow     : v1(100%, 응답O) + v2(복제, 응답 버림)
```

---

## 3. 전략별 상세 + 언제 쓰나

### Recreate

기존 버전을 **전부 종료한 뒤** 새 버전을 기동. 다운타임이 발생하지만 두 버전이 절대 공존하지 않음.

- **언제**: DB 스키마가 구/신 버전 동시 지원 불가, 라이선스상 단일 인스턴스만 허용, 야간 점검 가능한 내부 시스템.

### Rolling Update

파드를 한 묶음씩(`maxSurge`/`maxUnavailable` 기준) 점진 교체. K8s Deployment의 **기본 전략**.

- **장점**: 추가 리소스 최소, 별도 도구 불필요.
- **단점**: 배포 중 구/신 버전 공존(API 호환 필수), 세밀한 트래픽 비율 제어 불가, 롤백이 다시 롤링이라 느림.
- **언제**: 대부분의 일반 stateless 서비스 기본값.

### Blue-Green

운영(Blue)과 동일한 신규 환경(Green)을 통째로 띄우고, 검증 후 **라우터/서비스 셀렉터를 한 번에 전환**.

```yaml
# Service 셀렉터만 바꿔 트래픽 전환
spec:
  selector:
    app: myapp
    version: green   # blue → green 으로 변경하면 즉시 전환
```

- **장점**: 즉시 전환·즉시 롤백(셀렉터 원복), 전환 전 Green을 충분히 검증.
- **단점**: 리소스 2배, DB는 공유되므로 스키마 호환 여전히 필요.
- **언제**: 빠른 롤백이 절대 필요한 결제·핵심 API, 리소스 여유가 있을 때.

### Canary

신버전을 소수 인스턴스로 띄워 **트래픽 일부(5% → 25% → 50% → 100%)**만 보내며 메트릭으로 판정. 이상 시 즉시 회수.

- **장점**: 실제 트래픽으로 점진 검증, 블래스트 반경(영향 범위) 최소.
- **단점**: 트래픽 분배(서비스 메시/인그레스)와 메트릭 기반 자동 판정 도구 필요.
- **언제**: 사용자 영향이 큰 변경, 성능 회귀가 우려되는 릴리스.

### Shadow (Dark Launch)

실트래픽을 신버전으로 **복제 전송하되 응답은 버림**. 사용자에게 영향 없이 부하·정합성 검증.

- **언제**: 대규모 리팩터링, 신규 추론 서버 등 응답 검증이 필요 없는 읽기성 검증. 쓰기 경로는 부작용(이중 결제 등) 주의.

---

## 4. K8s 네이티브 배포

### RollingUpdate — maxSurge / maxUnavailable

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 목표 대비 추가로 띄울 수 있는 파드 (절대값/%)
      maxUnavailable: 0    # 동시에 내릴 수 있는 파드 — 0이면 무중단 보장
  minReadySeconds: 10      # readiness 통과 후 이 시간만큼 안정되어야 "사용 가능"
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: myapp
          image: myorg/myapp:2.0
```

| 파라미터 | 의미 | 권장 |
|---------|------|------|
| `maxSurge` | 추가 파드 허용량 | `1` 또는 `25%` (빠른 롤아웃 시 ↑) |
| `maxUnavailable` | 동시 중단 허용량 | **`0`** (무중단), 리소스 부족 시에만 ↑ |
| `minReadySeconds` | 안정 대기 시간 | `10~30s` (기동 직후 크래시 흡수) |

### Probe 순서 — startup → readiness → liveness

```
[기동 ~ 종료 라이프사이클]
  1) startupProbe   — 느린 부팅 완료까지 다른 probe 비활성 (JVM 워밍업 보호)
  2) readinessProbe — 통과 전까지 Service 엔드포인트에서 제외 (트래픽 차단)
  3) livenessProbe  — 실패 시 컨테이너 재시작 (데드락 등 자가 복구)

핵심: liveness 가 readiness 보다 먼저/공격적으로 돌면
      기동 중인 정상 파드를 죽이는 재시작 루프 발생 → startupProbe 로 보호
```

```yaml
startupProbe:                              # 부팅 보호: 30 × 5s = 150s 까지 허용
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5
readinessProbe:                            # 트래픽 게이트
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:                             # 자가 복구
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
```

> [!tip] Readiness Gate
> Pod의 `readinessGates`로 **외부 조건(LB 등록 완료 등)**을 추가 준비 조건에 포함시킬 수 있다. ALB/NLB가 타깃을 정상 등록할 때까지 파드를 Ready로 보지 않게 하여, 롤링 중 LB가 아직 라우팅하지 않는 파드로 트래픽이 가는 공백을 막는다.

```yaml
spec:
  readinessGates:
    - conditionType: target-health.elbv2.k8s.aws/<lb-name>
```

---

## 5. 점진 카나리 자동화 — Argo Rollouts / Flagger

K8s 기본 Deployment는 **트래픽 비율 기반 카나리를 직접 지원하지 않는다**. 별도 컨트롤러로 단계적 가중치·자동 판정을 구현한다.

| 도구 | 방식 | 특징 |
|------|------|------|
| **Argo Rollouts** | `Rollout` CRD가 Deployment 대체 | step 기반 가중치, AnalysisRun으로 메트릭 판정, GitOps(Argo CD) 친화 |
| **Flagger** | 기존 Deployment 감시 | 서비스 메시(Istio/Linkerd)·인그레스 연동, 메트릭 기반 자동 승급/롤백 |

```yaml
# Argo Rollouts — 단계적 카나리 + 메트릭 분석
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5          # 5% 트래픽
        - pause: { duration: 5m }
        - analysis:             # Prometheus 쿼리로 성공률 판정
            templates:
              - templateName: success-rate
        - setWeight: 25
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100        # 통과하면 전체 승급, 실패하면 자동 롤백
```

```yaml
# AnalysisTemplate — 카나리 판정 기준 (5xx 비율)
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: error-rate
      interval: 1m
      failureLimit: 2          # 2회 초과하면 카나리 실패 → 롤백
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_server_requests_seconds_count{status=~"5..",version="canary"}[2m]))
            / sum(rate(http_server_requests_seconds_count{version="canary"}[2m]))
```

---

## 6. DB 마이그레이션과의 순서 — Expand-Contract

배포 전략과 무관하게, **구/신 버전이 같은 스키마에서 공존**하는 순간이 반드시 생긴다(롤링·카나리·블루그린 전부). 따라서 스키마는 항상 **하위 호환**이어야 한다.

```
[Expand-Contract (= Parallel Change) 3단계]

  Expand   ① 스키마를 "추가만" — 새 컬럼/테이블 추가 (NULL 허용, 기본값)
           구버전·신버전 모두 동작 가능한 상태 유지

  Migrate  ② 신버전 앱 배포 — 새/구 컬럼 동시 쓰기(dual-write) 후 데이터 백필

  Contract ③ 구버전이 완전히 빠진 뒤 "제거" — 옛 컬럼 DROP, 제약 강화

[금지: 한 번에 하기]
  ❌ 컬럼 RENAME / NOT NULL 즉시 추가 / 컬럼 DROP 후 배포
     → 롤링 중 구버전이 깨짐, 롤백 불가
```

예: 컬럼 이름 변경(`name` → `full_name`)

1. **Expand** — `full_name` 컬럼 추가(NULL 허용). 마이그레이션은 앱 배포보다 **먼저**.
2. **Migrate** — 신버전은 두 컬럼 모두 쓰기, 읽기는 `full_name` 우선. 백필 스크립트로 기존 행 복사.
3. **Contract** — 구버전 트래픽 0 확인 후, 다음 릴리스에서 `name` DROP.

자세한 도구·롤백은 [[../data/DB-Migration]] 참고.

---

## 7. 롤백 절차

```bash
# Rolling (K8s 기본) — 리비전 히스토리 기반
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp                 # 직전 리비전으로
kubectl rollout undo deployment/myapp --to-revision=4 # 특정 리비전으로
kubectl rollout status deployment/myapp               # 롤백 진행 확인

# Blue-Green — 서비스 셀렉터 원복 (즉시)
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'

# Canary (Argo Rollouts) — 카나리 중단·트래픽 회수
kubectl argo rollouts abort myapp
kubectl argo rollouts undo myapp
```

```
[롤백 판단 기준 — 사전 정의 필수]
  - 5xx 에러율 > 임계 (예: 1%)        → 자동/즉시 롤백
  - P99 latency 회귀 (예: +50%)       → 롤백
  - 핵심 비즈니스 메트릭 급락(주문 등)  → 롤백
  ⚠ DB Contract(컬럼 DROP) 이후엔 앱만 롤백 불가 — Expand-Contract가 롤백 가능성을 지킴
```

---

## 8. 모니터링 연계 — 카나리 메트릭 판정

```
[자동 판정 루프 (Argo Rollouts / Flagger)]

  배포 → 카나리 N% → [대기] → Prometheus 쿼리 → 판정
                                   │
                  ┌────────────────┴────────────────┐
              성공률 OK                         임계 초과
                  │                                 │
            다음 가중치로 승급                   자동 롤백
```

판정에 쓰는 핵심 메트릭은 RED(Rate/Errors/Duration)이며, 카나리/베이스라인을 **label로 구분**해 비교한다.

```promql
# 카나리 vs 안정 버전 5xx 비율 비교 (Grafana 대시보드)
sum(rate(http_server_requests_seconds_count{status=~"5..",version="canary"}[2m]))
  / sum(rate(http_server_requests_seconds_count{version="canary"}[2m]))
# ↑ 가 stable 대비 유의하게 높으면 롤백
```

SLI/에러 버짓과 연계하면, 카나리 판정 임계를 **에러 버짓 소진율** 기준으로 둘 수도 있다. 구현은 [[../../../infra/observability/Prometheus-Grafana]] 참고.

---

## 관련
- [[Kubernetes]] — Deployment·probe·Service 기본
- [[CICD]] — 파이프라인에서 배포 전략 실행·승인 게이트
- [[../data/DB-Migration]] — Expand-Contract, Flyway/Liquibase 롤백
- [[../../../infra/observability/Prometheus-Grafana]] — 카나리 메트릭·판정 쿼리
- [[../resilience/Resilience4j]] — 배포 중 부분 장애 격리(서킷브레이커)
