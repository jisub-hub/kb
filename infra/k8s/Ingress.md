---
tags:
  - k8s
  - ingress
  - networking
  - nginx
created: 2026-06-15
---

# Ingress & Ingress Controller

> [!summary] 한 줄 요약
> **클러스터 외부 HTTP/HTTPS 트래픽을 내부 Service로 라우팅**하는 규칙 오브젝트. Service마다 LoadBalancer를 만드는 대신 Ingress 1개로 통합 관리. **공식적으로는 [[GatewayAPI]](GatewayClass/Gateway/HTTPRoute)로 이전이 권장**된다.

---

## 1. Ingress가 필요한 이유

```
LoadBalancer 방식: 서비스마다 클라우드 LB 생성
  → /api/orders → LB1 → order-svc
  → /api/users  → LB2 → user-svc
  → 비용 × 서비스 수

Ingress 방식: LB 1개 + Ingress 규칙
  → 외부 → LB1 → Ingress Controller → /api/orders → order-svc
                                     → /api/users  → user-svc
```

---

## 2. 구성 요소

```
외부 트래픽
    ↓
[LoadBalancer Service] ← 클라우드 LB (IP 1개)
    ↓
[Ingress Controller Pod] ← Nginx/Traefik/etc. 실제 프록시 역할
    ↓ (Ingress 규칙 읽어서)
[Ingress Object] ← 사용자가 정의하는 라우팅 규칙
    ↓
[ClusterIP Service → Pod]
```

- **Ingress Controller**: 실제로 트래픽을 처리하는 Pod (Nginx, Traefik, HAProxy 등). 클러스터에 별도 설치 필요.
- **Ingress Object**: 라우팅 규칙만 선언하는 오브젝트. Controller가 없으면 동작 안 함.

---

## 3. Nginx Ingress Controller 설치

```bash
# Helm으로 설치 (가장 일반적)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

---

## 4. Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    # Nginx 전용 설정
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # TLS 자동 발급
spec:
  ingressClassName: nginx    # 어떤 Controller를 사용할지

  # TLS 설정
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret    # 인증서 저장 Secret

  rules:
    - host: api.example.com
      http:
        paths:
          # 경로 기반 라우팅
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-svc
                port:
                  number: 80
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-svc
                port:
                  number: 80

    # 호스트 기반 라우팅
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-svc
                port:
                  number: 80
```

---

## 5. pathType 비교

| pathType | 동작 | 예시 |
|---|---|---|
| `Exact` | 정확히 일치 | `/api/orders` 만 매칭 |
| `Prefix` | 경로 접두사 일치 | `/api/orders` 와 `/api/orders/123` 모두 매칭 |
| `ImplementationSpecific` | Controller별 구현 | Regex 등 Controller가 결정 |

---

## 6. Ingress의 한계 → GatewayAPI로

| 한계 | 내용 |
|---|---|
| 표준화 부족 | 고급 기능은 Controller별 `annotations`에 의존 → 이식성 낮음 |
| 역할 분리 없음 | 인프라팀(GW 설정)과 앱팀(라우팅 규칙)이 분리 불가 |
| HTTP/HTTPS만 | TCP/UDP 라우팅 불가 |
| 표현력 제한 | 가중치 기반 트래픽 분할, 헤더 기반 라우팅 표준화 없음 |

> [!info] 공식 권장
> Kubernetes SIG-Network는 **[[GatewayAPI]]**(GatewayClass/Gateway/HTTPRoute)를 Ingress의 공식 후계로 지정했다. 신규 클러스터는 GatewayAPI 사용을 고려할 것.

---

## 7. 관련
- [[GatewayAPI]] · [[Service]] · [[Deployment]]
