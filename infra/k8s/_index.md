---
tags:
  - infra
  - kubernetes
  - k8s
  - moc
  - index
created: 2026-06-15
---

# Kubernetes MOC

> 컨테이너를 자동 배포·스케일링·복구하는 오케스트레이션 플랫폼. 선언적 YAML로 원하는 상태를 정의하면 k8s가 현실을 맞춘다.

## 파일 구성

### 클러스터 & 기초
- [[Kubernetes]] — 클러스터 아키텍처 (Control Plane / Worker Node / 컴포넌트)

### 워크로드 오브젝트
- [[Pod]] — 최소 실행 단위, Lifecycle, Probe, Resources
- [[ReplicaSet]] — 파드 수 유지, Deployment와의 관계
- [[Deployment]] — 선언적 배포, 롤링 업데이트, 롤백

### 네트워킹
- [[Service]] — ClusterIP · NodePort · LoadBalancer · Headless
- [[Ingress]] — Ingress Controller (Nginx), 라우팅 규칙
- [[GatewayAPI]] — GatewayClass · Gateway · HTTPRoute (공식 표준, Ingress 후계)

### 설정 & 스토리지
- [[ConfigMap-Secret]] — 환경변수·설정 분리, Secret 보안 관리
- [[PV-PVC]] — PersistentVolume · PVC · StorageClass · 동적 프로비저닝

### 스케줄링 & 스케일링
- [[Taint-Toleration]] — Taint · Toleration · NodeAffinity · 파드 배치 제어
- [[HPA]] — HorizontalPodAutoscaler · VPA · KEDA
- [[Resource-Management]] — requests/limits·QoS/eviction·CPU throttling·OOMKill·JVM 컨테이너 인식 ⭐

### 배포 전략 & GitOps
- [[RollingUpdate]] — RollingUpdate · Recreate · Blue-Green · Canary
- [[GitOps]] — ArgoCD · Flux · GitOps 패턴

### 패키지 관리
- [[Helm]] — Chart 구조, helm install/upgrade/rollback, Helmfile, CI/CD 연동
- [[Kustomize]] — base/overlay 구조, Strategic Merge Patch, images 태그 교체, kubectl -k

### 보안
- [[RBAC]] — Role/ClusterRole, ServiceAccount 최소권한, `kubectl auth can-i`
- [[Network-Policy]] — Default Deny, Pod 간 통신 격리, CNI(Cilium/Calico)
- [[Security-Context]] — runAsNonRoot, readOnlyRootFilesystem, capabilities, PSA

### 스테이트풀 & DB
- [[StatefulSet-DB]] — K8s에서 DB·Redis를 Pod로 운영하지 않는 이유, StatefulSet 설정 지옥
- [[CNPG]] — CloudNativePG 오퍼레이터: 자동 페일오버·백업/PITR·롤링 업그레이드·모니터링

### 자격증
- [[certification/Kubernetes-Certification]] — CKA(관리자), CKAD(개발자), CKS(보안), 도메인별 준비 전략·명령어·팁

## 핵심 오브젝트 관계
```
Deployment
  └─ manages ─→ ReplicaSet
                  └─ manages ─→ Pod (× replicas)
                                  └─ exposed by ─→ Service
                                                     └─ routed by ─→ Ingress / GatewayAPI
```

## 관련
- [[../container/Container]] · [[../ssh-cicd/GitLab-CICD]] · [[../terraform/Terraform]]
- 공식 문서: [kubernetes.io](https://kubernetes.io/docs/) · [helm.sh](https://helm.sh/docs/) · [kustomize.io](https://kustomize.io/)
