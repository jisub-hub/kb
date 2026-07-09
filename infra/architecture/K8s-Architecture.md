---
tags:
  - infra
  - architecture
  - kubernetes
  - k8s
  - container
created: 2026-06-15
---

# Kubernetes 기반 아키텍처

> [!summary] 한 줄 요약
> 3-Tier VM 기반에서 **컨테이너 오케스트레이션 플랫폼으로 전환**한 아키텍처. Deployment·Service·Ingress로 동일한 3계층 구조를 구현하되, 자동 스케일링·롤링 배포·자가 치유가 플랫폼 수준에서 제공된다.

---

## 1. VM 기반 vs k8s 기반 비교

| | VM 3-Tier | k8s 기반 |
|---|---|---|
| 배포 단위 | VM 인스턴스 | Pod (컨테이너) |
| 스케일링 | 수동 또는 Auto Scaling Group | HPA (자동, 메트릭 기반) |
| 장애 복구 | VM 재시작 (수 분) | Pod 자동 재생성 (수 초) |
| 배포 전략 | 수동 Rolling | RollingUpdate 자동 |
| 리소스 효율 | VM 낭비 가능 | Bin packing (최적 배치) |
| 설정 관리 | 서버 직접 설정 | ConfigMap/Secret |
| 서비스 디스커버리 | DNS, 수동 등록 | k8s Service 자동 |

---

## 2. 전체 아키텍처

```
인터넷
  ↓
[Cloud Load Balancer] (클라우드 관리형)
  ↓
[Ingress Controller / Gateway API]  ← k8s Ingress / HTTPRoute
  ↓  (URL 기반 라우팅)
[Service (ClusterIP)]
  ↓
[Pod 집합 — Deployment]
  Web Pods (Nginx)         ← replicas: 2~N, HPA 적용
  WAS Pods (Spring Boot)   ← replicas: 2~N, HPA 적용
  ↓
[Service (Headless)]
  ↓
[StatefulSet Pods]
  DB Primary/Standby       ← 또는 외부 관리형 DB
```

---

## 3. Namespace 분리 전략

```
namespace: production
  - web-deployment
  - was-deployment
  - configmap/secret

namespace: staging
  - web-deployment (replicas: 1)
  - was-deployment (replicas: 1)

namespace: monitoring
  - prometheus
  - grafana

namespace: infra
  - ingress-controller
  - cert-manager
  - argocd
```

---

## 4. 클라우드 통합 컴포넌트

```
[클라우드 Load Balancer]
        ↓
[Ingress / GatewayAPI]
        ↓
[Service → Pod]
        ↓
[PVC → Cloud Storage]          ← OCI Block Volume / AWS EBS
        ↓ (Service GW)
[클라우드 Object Storage]      ← OCI Object Storage / AWS S3
```

---

## 5. 고가용성 설계

**Pod 수준 HA:**
```yaml
# Deployment
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxUnavailable: 0    # 무중단

# Pod Anti-Affinity (다른 노드에 분산)
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - topologyKey: kubernetes.io/hostname
```

**노드 수준 HA:**
- Worker Node를 최소 3개, 여러 AZ에 분산 배치
- Control Plane도 Multi-AZ 구성 (관리형 k8s: EKS, OKE, AKS는 기본 HA)

---

## 6. 관찰성 (Observability) 스택

```
Pod → [로그] → Fluentd/Fluentbit Sidecar → Elasticsearch/CloudWatch
Pod → [메트릭] → Prometheus + Grafana Dashboard
Pod → [트레이스] → OpenTelemetry Agent → Jaeger/Tempo
```

---

## 7. GitOps 배포 흐름

```
개발자 코드 push
  → CI (GitLab/GitHub Actions): 이미지 빌드 → Registry 푸시
  → 이미지 태그 업데이트 commit → k8s 매니페스트 Git repo
  → ArgoCD/Flux 자동 감지 → kubectl apply
  → k8s: RollingUpdate 실행 → 구 Pod 교체
```

→ [[../k8s/GitOps]] 참고

---

## 8. k8s 도입 시점 기준

| 상황 | 권장 |
|---|---|
| 서비스 초기, 팀 소규모 | VM 3-Tier로 시작 |
| 서비스 수 증가 (5개 이상) | k8s 도입 고려 |
| 잦은 배포, 스케일링 필요 | k8s 도입 강력 권장 |
| 팀에 k8s 운영 경험 없음 | 관리형 서비스(EKS/OKE/AKS) + 학습 |

> [!warning] k8s 운영 오버헤드
> k8s 자체가 복잡한 시스템이다. 작은 팀이 단순 서비스를 운영할 때는 오버킬. **서비스 복잡도와 팀 역량이 갖춰진 후** 도입 권장.

---

## 9. 관련
- [[../k8s/_index]] · [[3-Tier-Architecture]] · [[MSA-Architecture]]
