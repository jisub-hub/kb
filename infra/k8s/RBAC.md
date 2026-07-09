---
tags:
  - k8s
  - rbac
  - security
  - kubernetes
created: 2026-06-16
---

# K8s RBAC — 역할 기반 접근 제어

> [!summary] 한 줄 요약
> K8s에서 **누가(Subject) 어떤 리소스(Resource)에 어떤 작업(Verb)을 할 수 있는지**를 Role/ClusterRole로 정의하고 RoleBinding/ClusterRoleBinding으로 연결한다. 기본은 **최소 권한 원칙**.

> [공식 문서](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 참고

---

## 1. RBAC 4대 오브젝트

```
Subject (누가)          Role/ClusterRole (무엇을)      Binding (연결)
────────────────        ─────────────────────────       ──────────────────────
User                    Role (네임스페이스 범위)  ─►    RoleBinding
Group                   ClusterRole (클러스터 전체) ─►  ClusterRoleBinding
ServiceAccount
```

| 오브젝트 | 범위 | 설명 |
|---------|------|------|
| `Role` | 특정 네임스페이스 | 해당 NS의 리소스만 제어 |
| `ClusterRole` | 클러스터 전체 | 모든 NS 또는 클러스터 수준 리소스 |
| `RoleBinding` | 특정 네임스페이스 | Role 또는 ClusterRole을 NS 범위로 바인딩 |
| `ClusterRoleBinding` | 클러스터 전체 | ClusterRole을 클러스터 범위로 바인딩 |

---

## 2. Role & RoleBinding

```yaml
# role.yaml — 특정 네임스페이스의 pod/log 조회 권한
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]                  # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
```

```yaml
# rolebinding.yaml — 사용자에게 Role 부여
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring          # 다른 NS의 SA도 가능
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 3. ClusterRole & ClusterRoleBinding

```yaml
# 노드 모니터링용 ClusterRole (클러스터 전체 접근 필요)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/metrics"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. ServiceAccount — 파드용 아이덴티티

```yaml
# ServiceAccount 생성
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: production
automountServiceAccountToken: false   # 불필요하면 토큰 자동 마운트 비활성화 ⭐
```

```yaml
# 파드에 SA 연결
spec:
  serviceAccountName: backend-sa
  automountServiceAccountToken: false  # SA 수준 설정을 파드에서도 재확인
```

```yaml
# SA에 최소 권한만 부여
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]                     # 읽기만, 쓰기 없음
    resourceNames: ["app-config"]      # 특정 리소스만 지정
```

---

## 5. 내장 ClusterRole 활용

```bash
# 자주 쓰는 내장 ClusterRole
kubectl get clusterroles | grep -E "admin|edit|view|cluster-admin"
```

| 내장 ClusterRole | 권한 수준 | 사용처 |
|----------------|----------|--------|
| `cluster-admin` | 최고 권한 | 운영자 (절대 서비스 계정에 부여 금지) |
| `admin` | 네임스페이스 전체 관리 | 팀 리더 (RoleBinding으로 NS 범위 제한) |
| `edit` | 대부분 리소스 생성/수정 | 개발자 |
| `view` | 읽기 전용 | CI/CD 조회, 모니터링 |

```yaml
# 기존 ClusterRole을 특정 NS 범위로만 바인딩 (ClusterRole + RoleBinding 조합)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-edit
  namespace: development
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole   # ClusterRole이지만 RoleBinding → NS 범위로 제한
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

---

## 6. 권한 검증

```bash
# 특정 사용자/SA의 권한 확인
kubectl auth can-i create pods --namespace production
kubectl auth can-i create pods --namespace production --as alice
kubectl auth can-i create pods --namespace production \
  --as system:serviceaccount:production:backend-sa

# SA의 전체 권한 목록 조회
kubectl auth can-i --list --namespace production \
  --as system:serviceaccount:production:backend-sa

# 특정 SA에 어떤 RoleBinding이 연결됐는지
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '.items[] | select(.subjects[]?.name=="backend-sa")'
```

---

## 7. 보안 원칙

1. **기본 SA 사용 금지** — 각 애플리케이션마다 전용 SA 생성
2. **automountServiceAccountToken: false** — API 서버에 접근 안 하는 파드는 반드시 비활성화
3. **cluster-admin 최소화** — 클러스터 관리자 계정도 namespace admin으로 충분한 경우 많음
4. **리소스 이름 지정** — `resourceNames: ["specific-secret"]`으로 접근 범위 최소화
5. **정기 감사** — `kubectl get rolebindings,clusterrolebindings -A`로 주기적 검토
6. **IRSA/Workload Identity** — 클라우드 서비스(S3, GCS 등) 접근 시 SA에 IAM Role 연결

---

## 8. 관련
- [[Kubernetes]] · [[Network-Policy]] · [[Security-Context]] · [[ConfigMap-Secret]]
- [[../security/Cloud-Security-Products]] · [[Web-Attacks]]
