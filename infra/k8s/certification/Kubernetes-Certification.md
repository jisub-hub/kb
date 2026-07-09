---
tags:
  - kubernetes
  - k8s
  - certification
  - cka
  - ckad
  - cks
  - cncf
created: 2026-06-16
---

# Kubernetes 자격증 — CKA · CKAD · CKS

> [!summary] 한 줄 요약
> **CKA**(관리자), **CKAD**(개발자), **CKS**(보안 전문가). 모두 CNCF가 주관하는 **실기 시험** (객관식 없음, 직접 kubectl 명령으로 과제 수행). CKS는 CKA 합격 후 응시 가능.

---

## 1. 세 자격증 비교

| 항목 | CKA | CKAD | CKS |
|------|-----|------|-----|
| **이름** | Certified Kubernetes Administrator | Certified Kubernetes Application Developer | Certified Kubernetes Security Specialist |
| **대상** | 클러스터 관리자 | 앱 개발자·DevOps | 보안 엔지니어 |
| **시험 시간** | 2시간 | 2시간 | 2시간 |
| **문제 수** | 15~20개 | 15~20개 | 15~20개 |
| **합격선** | 66% | 66% | 67% |
| **유효기간** | 3년 | 3년 | 2년 |
| **선수 조건** | 없음 | 없음 | **CKA 보유** |
| **시험 환경** | 원격 감독 브라우저 터미널 | 동일 | 동일 |
| **응시료** | $395 | $395 | $395 |

> 모든 시험은 **Open Book** — 공식 K8s 문서(`kubernetes.io/docs`, `kubernetes.io/blog`) 참조 가능.

---

## 2. CKA — Certified Kubernetes Administrator

### 시험 도메인 & 비중

```
Cluster Architecture, Installation & Configuration  25%
  - kubeadm으로 클러스터 설치/업그레이드
  - etcd 백업·복구
  - RBAC 설정
  - 인증서 관리

Workloads & Scheduling                               15%
  - Deployment, ReplicaSet 관리
  - ConfigMap, Secret
  - Pod 스케줄링 (NodeSelector, Affinity, Taint/Toleration)
  - Resource Requests/Limits

Services & Networking                                20%
  - Service (ClusterIP, NodePort, LoadBalancer)
  - Ingress 설정
  - CoreDNS 트러블슈팅
  - NetworkPolicy

Storage                                              10%
  - PersistentVolume, PVC
  - StorageClass
  - Volume 마운트

Troubleshooting                                      30%
  - 컨트롤 플레인 컴포넌트 장애 복구
  - Worker 노드 장애
  - 애플리케이션 장애 (CrashLoopBackOff, Pending, ImagePullBackOff)
  - 로그 분석 (kubectl logs, describe, events)
```

### 핵심 준비 포인트

```bash
# etcd 백업 (반드시 외워야 함)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# etcd 복구
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# 클러스터 업그레이드 (kubeadm)
apt-get update && apt-get install -y kubeadm=1.29.0-00
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0

# Control Plane 인증서 갱신
kubeadm certs renew all
kubeadm certs check-expiration

# Pod 강제 재배치 (노드 drain)
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
# 작업 후 복구
kubectl uncordon node01

# kubeconfig 컨텍스트 전환 (다중 클러스터)
kubectl config use-context cluster1
kubectl config get-contexts
```

---

## 3. CKAD — Certified Kubernetes Application Developer

### 시험 도메인 & 비중

```
Application Design & Build                        20%
  - 컨테이너 이미지 빌드 (Dockerfile)
  - 멀티 컨테이너 Pod (sidecar, init container)
  - Job, CronJob

Application Deployment                            20%
  - Deployment, Rolling Update, Rollback
  - Helm 차트 기본 사용
  - Kustomize 오버레이

Application Observability & Maintenance          15%
  - Liveness/Readiness/Startup Probe
  - kubectl logs, exec, port-forward
  - API 버전 확인 및 오브젝트 이전

Application Environment, Configuration & Security 25%
  - ConfigMap, Secret 주입 (env/volume)
  - SecurityContext (runAsUser, readOnlyRootFilesystem)
  - ServiceAccount, RBAC (Pod 레벨)
  - Resource Requests/Limits

Services & Networking                            20%
  - Service 생성 및 DNS 확인
  - Ingress 규칙 작성
  - NetworkPolicy
```

### 핵심 준비 포인트

```bash
# 빠른 리소스 생성 (--dry-run으로 YAML 생성 후 편집)
kubectl create deployment nginx --image=nginx --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

kubectl create service clusterip my-svc --tcp=80:8080 \
  --dry-run=client -o yaml

kubectl create configmap app-config --from-literal=ENV=prod \
  --from-file=config.properties

# Job 생성
kubectl create job data-processor --image=busybox \
  -- sh -c 'echo Processing; sleep 5'

# CronJob
kubectl create cronjob cleanup --image=busybox \
  --schedule='0 2 * * *' -- sh -c 'rm -rf /tmp/*'

# Pod 내 명령 실행
kubectl exec -it pod-name -- /bin/bash
kubectl exec pod-name -c container-name -- env

# 포트 포워딩으로 로컬 테스트
kubectl port-forward svc/my-service 8080:80

# init container 템플릿
kubectl run main-pod --image=nginx --dry-run=client -o yaml > pod.yaml
# initContainers 섹션 수동 추가
```

---

## 4. CKS — Certified Kubernetes Security Specialist

### 시험 도메인 & 비중

```
Cluster Setup                                    10%
  - CIS Benchmark (kube-bench)
  - Ingress TLS 설정
  - 노드 메타데이터 보호
  - GUI 대시보드 최소화

Cluster Hardening                                15%
  - RBAC 최소 권한 원칙
  - ServiceAccount 토큰 마운트 비활성화
  - K8s API 업그레이드 (최신 패치)
  - 어드미션 컨트롤러

System Hardening                                 15%
  - OS 레벨 보안 (AppArmor, Seccomp)
  - IAM (AWS/GCP 서비스 계정)
  - 불필요한 커널 모듈 비활성화
  - 파일/디렉토리 권한

Minimize Microservice Vulnerabilities            20%
  - Security Context (runAsNonRoot, capabilities drop)
  - Pod Security Standards (PSA)
  - OPA Gatekeeper / Kyverno 정책
  - 시크릿 관리 (Vault 통합)

Supply Chain Security                            20%
  - 이미지 스캔 (Trivy)
  - 이미지 서명 (Cosign)
  - 레지스트리 화이트리스트
  - 취약한 이미지 기반 Pod 제거

Monitoring, Logging & Runtime Security          20%
  - Falco (런타임 위협 탐지)
  - 감사 로그 (Audit Policy)
  - Behavioral Analytics
  - 컨테이너 이뮤터블 파일시스템
```

### 핵심 준비 포인트

```bash
# kube-bench (CIS Benchmark 검사)
docker run --pid=host --userns=host --cap-add=audit_write \
  -v /etc:/etc:ro -v /var:/var:ro -v /proc:/proc:ro \
  aquasec/kube-bench:latest

# Trivy 이미지 취약점 스캔
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL myapp:1.0

# Falco 룰 예시 (컨테이너에서 bash 실행 감지)
- rule: Terminal Shell in Container
  desc: A shell was used as the entrypoint/exec point in a container
  condition: container.id != host and proc.name = bash
  output: "Shell opened in container (user=%user.name container=%container.name)"
  priority: WARNING

# AppArmor 프로필 적용
kubectl annotate pod mypod \
  container.apparmor.security.beta.kubernetes.io/mycontainer=localhost/my-profile

# Seccomp 프로필 적용
securityContext:
  seccompProfile:
    type: RuntimeDefault      # 기본 컨테이너 런타임 Seccomp

# 감사 로그 정책 (audit-policy.yaml)
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata         # 요청 메타데이터만 (RequestBody 제외)
    resources:
      - group: ""
        resources: ["secrets"]    # Secret 접근 기록
  - level: None             # 나머지 무시
    users: ["system:kube-proxy"]
```

---

## 5. 시험 환경 & 전략

### 공통 준비

```bash
# 시험 시작 전 반드시 설정
alias k=kubectl
export do="--dry-run=client -o yaml"   # 빠른 YAML 생성
export now="--force --grace-period 0"  # 즉시 삭제

# 자동 완성 설정 (시험장에서도 가능)
source <(kubectl completion bash)
complete -F __start_kubectl k

# 컨텍스트 확인 (문제마다 다른 클러스터)
kubectl config current-context
kubectl config use-context k8s-cluster1

# 빠른 도움말
kubectl explain pod.spec.containers.securityContext
kubectl explain deployment.spec.strategy
```

### 시험 팁

```
1. 시간 관리
   - 각 문제에 배점(%)이 표시됨
   - 모르는 문제는 플래그 후 나중에 (4점 문제에 20분 쓰면 손해)
   - 쉬운 문제부터: kubectl 명령으로 바로 해결되는 것 먼저

2. 문서 활용
   - kubernetes.io/docs 북마크 구성
   - Tasks > Configure Pods > 설정 예시 복붙
   - 검색: "site:kubernetes.io networkpolicy example"

3. YAML 생성 전략
   - kubectl create ... --dry-run=client -o yaml > file.yaml
   - vi file.yaml로 편집
   - kubectl apply -f file.yaml
   - 편집기: vi 기본, vim ~/.vimrc에 set expandtab ts=2 sw=2 설정

4. 검증 습관
   - kubectl get pods -n namespace (상태 확인)
   - kubectl describe pod <name> (이벤트, 에러 확인)
   - kubectl logs <pod> (앱 로그)
```

---

## 6. 추천 학습 순서

```
[CKA 준비 로드맵 (6~8주)]
  Week 1-2: K8s 아키텍처, Pod/Deployment/Service 기초
  Week 3:   kubeadm 설치, etcd 백업·복구, 인증서
  Week 4:   RBAC, NetworkPolicy, Storage (PV/PVC)
  Week 5:   Troubleshooting 집중 (가장 높은 배점 30%)
  Week 6-8: Killer.sh 모의시험 × 2회 (합격권 시뮬레이션)

[CKAD 준비 로드맵 (4~6주)]
  Week 1-2: Deployment, ConfigMap/Secret, 리소스 관리
  Week 3:   Security Context, ServiceAccount, Probe
  Week 4:   Helm, Kustomize, Job/CronJob
  Week 5-6: Killer.sh 모의시험

[CKS 준비 로드맵 (6~8주, CKA 보유 전제)]
  Week 1-2: RBAC 심화, PSA, OPA Gatekeeper
  Week 3:   Falco, 감사 로그, 이미지 보안
  Week 4:   AppArmor, Seccomp, Trivy
  Week 5-8: Killer.sh CKS 모의시험

[추천 자료]
  강의: Mumshad Mannambeth (Udemy) — 업계 표준
  모의시험: killer.sh (시험 구매 시 2회 포함)
  실습: KillerCoda, Play with Kubernetes
```

---

## 7. 관련
- [[../Kubernetes]] — 클러스터 아키텍처 기초
- [[../RBAC]] — Role/ClusterRole 상세
- [[../Security-Context]] — runAsNonRoot, capabilities
- [[../Network-Policy]] — NetworkPolicy 작성
- [[../PV-PVC]] — 스토리지 구성
