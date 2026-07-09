---
tags:
  - infra
  - kubernetes
  - k8s
  - container
created: 2026-06-15
---

# Kubernetes (k8s)

> [!summary] 한 줄 요약
> 컨테이너를 **자동 배포·스케일링·복구**하는 오케스트레이션 플랫폼. 선언적 YAML로 원하는 상태를 정의하면 k8s가 현실을 맞춘다.

---

## 1. 아키텍처

```
┌─────────────────── Control Plane ───────────────────┐
│  kube-apiserver  etcd  kube-scheduler  kube-controller-manager │
└──────────────────────────────────────────────────────┘
         │ (watch / reconcile)
┌────── Worker Node ──────┐   ┌────── Worker Node ──────┐
│  kubelet  kube-proxy    │   │  kubelet  kube-proxy    │
│  ┌─Pod─┐  ┌─Pod─┐      │   │  ┌─Pod─┐               │
│  │ C1  │  │ C2  │      │   │  │ C3  │               │
└─────────────────────────┘   └─────────────────────────┘
```

| 컴포넌트 | 역할 |
|---------|------|
| **kube-apiserver** | 모든 요청의 진입점. kubectl/다른 컴포넌트가 통신 |
| **etcd** | 클러스터 상태 저장 (분산 KV store) |
| **kube-scheduler** | 파드를 어느 노드에 배치할지 결정 |
| **kube-controller-manager** | Deployment·ReplicaSet 등 상태 유지 루프 |
| **kubelet** | 노드에서 파드 실행·상태 보고 |
| **kube-proxy** | 서비스 → 파드 네트워크 라우팅 |

---

## 2. 핵심 오브젝트

### Pod — 최소 실행 단위
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
    - name: myapp
      image: myorg/myapp:1.0
      ports:
        - containerPort: 8080
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
      resources:
        requests: { cpu: "250m", memory: "256Mi" }
        limits:   { cpu: "500m", memory: "512Mi" }
```

### Deployment — 파드 선언적 관리 + 롤링 업데이트
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myorg/myapp:1.0
          readinessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
            initialDelaySeconds: 10
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
```

### Service — 파드 접근 추상화
```yaml
# ClusterIP (클러스터 내부)
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP   # ClusterIP | NodePort | LoadBalancer
```

| 타입 | 접근 범위 | 사용처 |
|------|----------|--------|
| `ClusterIP` | 클러스터 내부만 | 서비스 간 통신 |
| `NodePort` | 노드 IP + 포트 | 개발/테스트 |
| `LoadBalancer` | 외부 LB 생성 | 클라우드 프로덕션 |

### ConfigMap & Secret
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: INFO
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_PASSWORD: "supersecret"   # base64 인코딩 자동
```

```yaml
# 파드에서 사용
envFrom:
  - configMapRef: { name: app-config }
  - secretRef:    { name: db-secret }
```

### Ingress — 외부 HTTP 라우팅
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service: { name: order-svc, port: { number: 80 } }
          - path: /users
            pathType: Prefix
            backend:
              service: { name: user-svc, port: { number: 80 } }
```

---

## 3. 자동 스케일링 — HPA

> [공식 문서](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 참고

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

---

## 4. Namespace — 논리적 격리

```bash
kubectl create namespace staging
kubectl apply -f deployment.yaml -n staging
kubectl get pods -n staging

# 기본 namespace 변경
kubectl config set-context --current --namespace=staging
```

---

## 5. 주요 kubectl 명령어

```bash
# 조회
kubectl get pods -A                         # 전체 namespace 파드
kubectl get pod myapp-xxx -o yaml           # YAML 상세
kubectl describe pod myapp-xxx              # 이벤트·상태 상세

# 배포
kubectl apply -f manifest.yaml              # 생성/업데이트
kubectl rollout status deployment/myapp     # 롤아웃 진행 확인
kubectl rollout undo deployment/myapp       # 이전 버전으로 롤백

# 디버깅
kubectl logs -f myapp-xxx                   # 로그 스트리밍
kubectl logs -f myapp-xxx -c sidecar        # 특정 컨테이너
kubectl exec -it myapp-xxx -- bash          # 컨테이너 접속
kubectl port-forward svc/myapp-svc 8080:80  # 로컬 포트 포워딩

# 리소스 정리
kubectl delete -f manifest.yaml
kubectl delete pod myapp-xxx --force
```

---

## 6. RBAC — 접근 제어

> [공식 문서](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 참고

```
[RBAC 구성 요소]
  ServiceAccount   → 파드의 신원 (=누구인가)
  Role             → 특정 namespace 내 권한 규칙
  ClusterRole      → 클러스터 전체 범위 권한
  RoleBinding      → ServiceAccount ↔ Role 연결
  ClusterRoleBinding → ServiceAccount ↔ ClusterRole 연결
```

```yaml
# 1. ServiceAccount 생성
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
---
# 2. Role — production namespace 내 Pod 읽기 전용
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
---
# 3. RoleBinding — SA와 Role 연결
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Deployment에 ServiceAccount 지정
spec:
  template:
    spec:
      serviceAccountName: myapp-sa

# 권한 확인 (can-i)
kubectl auth can-i list pods --as=system:serviceaccount:production:myapp-sa -n production
kubectl auth can-i delete deployments --as=system:serviceaccount:production:myapp-sa -n production
# → no

# 전체 권한 목록 조회
kubectl get rolebindings,clusterrolebindings -A | grep myapp-sa
```

### 최소 권한 원칙 적용

```yaml
# ❌ 나쁜 예: cluster-admin 부여 (흔한 실수)
roleRef:
  kind: ClusterRole
  name: cluster-admin    # 전능 권한 — 절대 앱에 부여 금지

# ✅ 좋은 예: 필요한 리소스·동사만 명시
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]        # 읽기만 필요하면 list·watch 도 제거
    resourceNames: ["app-config"]  # 특정 이름만 허용
```

---

## 7. Network Policy — 파드 간 통신 격리

> [공식 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/) 참고

기본 K8s는 **모든 파드가 서로 통신 가능**. Network Policy로 화이트리스트 기반 격리 적용. (CNI가 Cilium·Calico·Weave 등 NetworkPolicy 지원해야 작동)

```yaml
# Default Deny All — namespace 진입 트래픽 전부 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}        # 해당 namespace 모든 파드 대상
  policyTypes:
    - Ingress
```

```yaml
# order-service → payment-service 포트 8080만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-to-payment
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment        # 이 정책이 적용될 파드 (목적지)
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: order  # 허용할 출발지 파드
      ports:
        - protocol: TCP
          port: 8080
```

```yaml
# 외부 DB 접근 허용 (Egress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.1.0/24    # DB 서버 IP 대역
      ports:
        - protocol: TCP
          port: 5432
    - to:                         # DNS 허용 (없으면 도메인 해석 불가)
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

```bash
# 적용 확인
kubectl get networkpolicies -n production
kubectl describe networkpolicy allow-order-to-payment -n production

# 테스트: 허용 안 된 파드에서 접근 시도
kubectl exec -n production order-pod -- curl payment-svc:8080    # ✅ 연결됨
kubectl exec -n production admin-pod -- curl payment-svc:8080   # ❌ 차단됨
```

---

## 8. PersistentVolume / PVC

> [공식 문서](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 참고

```yaml
# StorageClass — 동적 프로비저닝 (클라우드 환경)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Retain       # Delete(기본) vs Retain(데이터 보존)
allowVolumeExpansion: true  # 온라인 볼륨 확장 허용
---
# PVC — 파드가 볼륨을 요청하는 방법
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce   # RWO: 단일 노드 읽쓰 | RWX: 다중 노드 읽쓰(NFS)
  resources:
    requests:
      storage: 20Gi
```

```yaml
# Deployment에서 PVC 사용
volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: app-data
containers:
  - name: myapp
    volumeMounts:
      - name: data-volume
        mountPath: /data
```

---

## 9. 베스트 프랙티스

- **resources.requests/limits 항상 지정** — 스케줄러 판단·OOM 방지
- **readinessProbe** 필수 — 준비 안 된 파드에 트래픽 차단
- **ConfigMap/Secret 분리** — 이미지에 설정 포함 ❌
- **Namespace 분리** — dev / staging / prod 환경 격리
- **롤링 업데이트 전략** — maxUnavailable: 0 으로 무중단 보장
- **Image 태그 latest 금지** — 명시적 버전 태그 사용

---

## 관련
- [[RBAC]] · [[Network-Policy]] · [[Security-Context]] — 보안 상세
- [[HPA]] · [[Taint-Toleration]] — 스케일링·스케줄링
- [[StatefulSet-DB]] — DB를 K8s에서 운영하면 안 되는 이유
- [[Helm]] — Chart 기반 패키지 관리 (Helmfile 포함)
- [[Kustomize]] — 템플릿 없는 YAML 오버레이 방식
- [[Container]] · [[Terraform]] · [[infra/_index]]
