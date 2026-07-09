---
tags:
  - k8s
  - network
  - security
  - networkpolicy
created: 2026-06-16
---

# K8s Network Policy — Pod 네트워크 격리

> [!summary] 한 줄 요약
> K8s 기본은 **모든 Pod 간 통신 허용**. NetworkPolicy로 Ingress/Egress 규칙을 선언하면 허용된 트래픽만 흐른다. **CNI 플러그인(Cilium, Calico)이 반드시 설치돼 있어야 동작**.

> [공식 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/) 참고

---

## 1. 왜 필요한가

```
[기본 상태 — no NetworkPolicy]
frontend ──► backend ──► database
frontend ──►────────────► database  ← 직접 접근 가능 (보안 위협)
외부 공격자 → 아무 pod → 다른 pod → DB  ← 횡이동(lateral movement) 가능

[NetworkPolicy 적용 후]
frontend ──► backend ──► database   ← 허용 경로만 통과
frontend ──►────────────► database  ← 차단
```

---

## 2. CNI 선택

NetworkPolicy API는 K8s 표준이지만 **실제 집행은 CNI가 한다.**

| CNI | 특징 | 권장 환경 |
|-----|------|----------|
| **Cilium** | eBPF 기반, L7 정책(HTTP path), 고성능 | GKE, EKS, 신규 클러스터 ⭐ |
| **Calico** | 성숙한 BGP 통합, L3/L4 정책 | 온프레미스, 네트워크 제어 중요 시 |
| **Weave** | 단순 설정 | 소규모 |
| Flannel | NetworkPolicy 미지원 | 정책 불필요한 환경 |

---

## 3. Default Deny (기본 차단 먼저)

```yaml
# 네임스페이스의 모든 Ingress 기본 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # 빈 selector = 네임스페이스 모든 Pod
  policyTypes:
    - Ingress              # Ingress 규칙 없음 = 모두 차단

---
# 모든 Egress도 차단 (더 엄격한 환경)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

> **순서**: 항상 default-deny 먼저 적용하고, 필요한 경로만 허용 규칙으로 추가.

---

## 4. 허용 규칙 예시

### 4.1. Frontend → Backend만 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend              # 이 정책의 대상 Pod
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend     # frontend label Pod에서만 허용
      ports:
        - protocol: TCP
          port: 8080
```

### 4.2. Backend → Database만 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

### 4.3. 네임스페이스 간 통신 허용

```yaml
# monitoring NS의 Prometheus가 production NS의 메트릭 scrape 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring   # NS 라벨로 선택
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 8080    # /actuator/prometheus 엔드포인트
```

### 4.4. DNS 쿼리 허용 (Egress 차단 시 필수)

```yaml
# default-deny-all 적용 후 DNS는 반드시 허용해야 서비스 디스커버리 동작
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## 5. 전체 아키텍처 예시

```
[internet]
    │
[Ingress Controller] (ingress NS)
    │ TCP 8080
[frontend] (production NS) ──TCP 8080──► [backend] ──TCP 8080──► [cache/redis]
                                              │ TCP 5432
                                          [database]

[prometheus] (monitoring NS) ──TCP 8080──► [모든 앱] (scrape)
```

```bash
# 위 구조 검증
kubectl exec -n production frontend-pod -- nc -zv backend-svc 8080   # 성공
kubectl exec -n production frontend-pod -- nc -zv database-svc 5432  # 차단 확인
```

---

## 6. 정책 확인 & 디버깅

```bash
# 네임스페이스의 모든 NetworkPolicy 조회
kubectl get networkpolicy -n production

# 특정 정책 상세
kubectl describe networkpolicy allow-frontend-to-backend -n production

# Cilium: 정책 적용 상태 확인
kubectl exec -n kube-system cilium-xxxx -- cilium policy get
kubectl exec -n kube-system cilium-xxxx -- cilium endpoint list

# Calico: 정책 확인
calicoctl get networkpolicy -n production
```

---

## 7. 관련
- [[RBAC]] · [[Kubernetes]] · [[Security-Context]]
- [[../network/Network-Basics]] · [[../security/Cloud-Security-Products]]
