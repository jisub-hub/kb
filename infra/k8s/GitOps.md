---
tags:
  - k8s
  - gitops
  - argocd
  - flux
  - cicd
created: 2026-06-15
---

# GitOps — ArgoCD & Flux

> [!summary] 한 줄 요약
> **Git 저장소가 클러스터 상태의 단일 진실 공급원(Single Source of Truth)**. Git에 커밋하면 ArgoCD/Flux가 자동으로 클러스터에 반영한다. CI/CD의 CD 단계를 Git 기반으로 선언적으로 관리.

---

## 1. GitOps 원칙

1. **선언적(Declarative)**: 클러스터 상태를 YAML로 선언
2. **버전 관리**: 모든 변경이 Git에 커밋으로 기록
3. **자동 동기화**: Operator가 Git과 클러스터 상태를 지속적으로 일치
4. **자동 복구**: 수동 변경 감지 → 자동으로 Git 상태로 되돌림

```
개발자 → git push → Git repo (YAML 변경)
                       ↓
                  ArgoCD/Flux가 감지
                       ↓
                  kubectl apply 자동 실행
                       ↓
                  클러스터 상태 = Git 상태
```

---

## 2. CI/CD에서 GitOps 위치

```
[CI] 코드 빌드 → Docker 이미지 빌드 → Registry 푸시
                                           ↓
[GitOps 트리거] 이미지 태그 업데이트 → Git commit (k8s 매니페스트 repo)
                                           ↓
[CD] ArgoCD/Flux → 클러스터에 자동 배포
```

- **CI**: GitLab CI, GitHub Actions (이미지 빌드·푸시)
- **CD**: ArgoCD/Flux (Git → 클러스터 자동 동기화)

---

## 3. ArgoCD

CNCF 졸업 프로젝트. GUI 대시보드가 강점. **Pull 방식** (ArgoCD가 Git을 주기적으로 polling).

### 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# 포트포워딩으로 UI 접근
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Application 정의

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default

  # Git 소스
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main        # 브랜치/태그/커밋
    path: apps/myapp/overlays/production

  # 배포 대상 클러스터/네임스페이스
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # 동기화 정책
  syncPolicy:
    automated:
      prune: true       # Git에서 삭제된 리소스도 클러스터에서 삭제
      selfHeal: true    # 수동 변경 감지 시 자동 Git 상태로 복구
    syncOptions:
      - CreateNamespace=true
```

### 주요 개념

| 개념 | 설명 |
|---|---|
| **App of Apps** | Application이 다른 Application들을 관리하는 패턴 |
| **Sync** | Git과 클러스터 상태를 일치시키는 작업 |
| **OutOfSync** | Git과 클러스터 상태가 다른 상태 |
| **Drift** | 수동 변경으로 Git과 클러스터가 벗어난 상태 |
| **Project** | Application을 그룹화하고 접근 권한 제한하는 단위 |

---

## 4. Flux v2

CNCF 졸업 프로젝트. CLI 중심, Helm/Kustomize 통합 강점. **Push 방식도 지원**.

### 설치

```bash
# Flux CLI 설치
brew install fluxcd/tap/flux

# 클러스터에 Flux 부트스트랩 (GitHub 연동)
flux bootstrap github \
  --owner=myorg \
  --repository=k8s-manifests \
  --branch=main \
  --path=clusters/production \
  --personal
```

### GitRepository + Kustomization

```yaml
# Git 소스 정의
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp-repo
  namespace: flux-system
spec:
  interval: 1m             # 1분마다 Git 폴링
  url: https://github.com/myorg/k8s-manifests
  ref:
    branch: main
---
# 클러스터에 적용할 경로
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: "./apps/production"
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp-repo
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
```

### HelmRelease — Helm Chart 자동 배포

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nginx-ingress
  namespace: infra
spec:
  interval: 5m
  chart:
    spec:
      chart: ingress-nginx
      version: "4.x.x"
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
  values:
    controller:
      replicaCount: 2
```

---

## 5. ArgoCD vs Flux 비교

| | ArgoCD | Flux v2 |
|---|---|---|
| UI | ✅ 웹 대시보드 | ❌ CLI 중심 (Weave GitOps로 보완) |
| 동기 방식 | Pull (polling) | Pull (기본), Push 지원 |
| Helm 지원 | ✅ | ✅ (HelmRelease) |
| Kustomize | ✅ | ✅ (Kustomization) |
| 멀티클러스터 | ✅ 강력 | ✅ |
| RBAC | ✅ 세밀한 권한 | ✅ (k8s RBAC 활용) |
| 알림 | Plugin 기반 | Notification Controller |
| 학습 곡선 | 낮음 (UI) | 중간 (CRD 중심) |
| 적합한 경우 | UI 선호, 빠른 시작 | Helm 집중, 코드 기반 |

---

## 6. 매니페스트 저장소 구조 (예시)

```
k8s-manifests/
├── apps/
│   ├── myapp/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── staging/
│   │       │   ├── kustomization.yaml   # base + staging 패치
│   │       │   └── patch-replicas.yaml
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           └── patch-replicas.yaml
└── clusters/
    ├── staging/          ← Flux/ArgoCD가 이 경로 감시
    └── production/
```

---

## 7. 관련
- [[Deployment]] · [[RollingUpdate]] · [[../ssh-cicd/GitLab-CICD]]
