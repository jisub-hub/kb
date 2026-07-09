---
tags:
  - k8s
  - deployment
  - rolling-update
  - blue-green
  - canary
created: 2026-06-15
---

# 배포 전략 — RollingUpdate · Recreate · Blue-Green · Canary

> [!summary] 한 줄 요약
> 배포 전략은 **가용성·롤백 속도·리소스 비용의 트레이드오프**. k8s 기본 전략은 RollingUpdate이며, Blue-Green/Canary는 Service/Ingress 조합이나 Argo Rollouts로 구현한다.

---

## 1. k8s 네이티브 전략 (Deployment spec.strategy)

### RollingUpdate (기본)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 추가 생성 허용 수 (숫자 또는 %)
      maxUnavailable: 0     # 동시 종료 허용 수 (0 = 무중단 필수)
```

```
replicas=3, maxSurge=1, maxUnavailable=0

Step1: v1 v1 v1       → v2 추가 → v1 v1 v1 v2 (4개)
Step2: v2 Ready 확인  → v1 하나 제거 → v1 v1 v2 (3개)
Step3: v1 v2 v2 → v2 v2 v2 완료
```

- **무중단 배포 가능** (maxUnavailable=0)
- 배포 중 v1/v2 혼재 → 버전 간 API 호환성 필요
- 문제 발견 시 `kubectl rollout undo`로 즉시 롤백

---

### Recreate

```yaml
spec:
  strategy:
    type: Recreate
```

```
Step1: v1 v1 v1 → 전체 삭제
Step2: (다운타임)
Step3: v2 v2 v2 생성
```

- **다운타임 발생** → 프로덕션 웹서버엔 부적합
- 버전 혼재가 절대 불가한 경우 (DB 스키마 변경과 동시 배포 등)
- 개발/스테이징 환경

---

## 2. 고급 전략 (Ingress/Service 또는 Argo Rollouts)

### Blue-Green 배포

```
[현재: Blue (v1)] ← 모든 트래픽
[새로운: Green (v2)] ← 준비 중

전환:
  Service selector: app=myapp-blue → app=myapp-green
  → 순간적으로 모든 트래픽이 v2로 전환
```

**Ingress 기반 구현:**

```yaml
# Blue Deployment (현재)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: myapp
          image: myorg/myapp:1.0

# Green Deployment (신버전)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: myapp
          image: myorg/myapp:2.0

# Service (selector 변경으로 Blue↔Green 전환)
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
    version: blue   # ← green으로 변경하면 즉시 전환
  ports:
    - port: 80
      targetPort: 8080
```

| 항목 | 내용 |
|---|---|
| 전환 방식 | Service selector 변경 → 즉각 전환 |
| 롤백 | selector를 blue로 되돌리면 즉시 롤백 |
| 리소스 | 2배 필요 (Blue+Green 동시 운영) |
| 버전 혼재 | 없음 (원자적 전환) |

---

### Canary 배포

소수 트래픽만 신버전에 보내 검증 후 점진적 확대.

**Ingress weight 기반 (Gateway API HTTPRoute):**

```yaml
# HTTPRoute (10% → canary, 90% → stable)
spec:
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: myapp-stable
          port: 80
          weight: 90
        - name: myapp-canary
          port: 80
          weight: 10
```

**헤더 기반 Canary (특정 사용자만 신버전 라우팅):**

```yaml
rules:
  - matches:
      - headers:
          - name: X-Canary
            value: "true"
    backendRefs:
      - name: myapp-canary
        port: 80
  - backendRefs:   # 기본 (나머지 전체)
      - name: myapp-stable
        port: 80
```

---

## 3. Argo Rollouts — 고급 배포 자동화

k8s 네이티브 한계를 넘어 **자동 분석·자동 프로모션·자동 롤백**을 제공하는 CNCF 프로젝트.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 10       # 1단계: 10% 트래픽
        - pause: { duration: 5m }  # 5분 관찰
        - setWeight: 30       # 2단계: 30%
        - pause: { duration: 10m }
        - setWeight: 100      # 전체 전환
      # 자동 분석 (Prometheus 메트릭)
      analysis:
        templates:
          - templateName: error-rate
        startingStep: 1
        args:
          - name: service-name
            value: myapp-canary
```

---

## 4. 전략 비교

| 전략 | 다운타임 | 롤백 속도 | 리소스 추가 | 버전 혼재 | 적합한 상황 |
|---|---|---|---|---|---|
| **RollingUpdate** | 없음 | 느림 (역방향 롤링) | 최소 | 있음 | 일반 무중단 배포 |
| **Recreate** | 있음 | 빠름 | 없음 | 없음 | 스키마 변경, 개발환경 |
| **Blue-Green** | 없음 | 즉시 | 2배 | 없음 | 큰 변경, 빠른 롤백 필요 |
| **Canary** | 없음 | 빠름 | 소량 | 있음 | 점진적 검증, 위험 최소화 |

---

## 5. 관련
- [[Deployment]] · [[Ingress]] · [[GatewayAPI]] · [[GitOps]]
