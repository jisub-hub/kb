---
tags:
  - k8s
  - gateway-api
  - networking
  - httproute
created: 2026-06-15
---

# Kubernetes Gateway API

> [!summary] 한 줄 요약
> **Ingress의 공식 후계 표준**. GatewayClass(인프라) · Gateway(진입점) · HTTPRoute(라우팅 규칙)를 분리해 역할별 권한을 나눌 수 있다. Kubernetes SIG-Network 공식 추천.

---

## 1. Ingress vs Gateway API

| | Ingress | Gateway API |
|---|---|---|
| 표준화 | annotations에 의존 (Controller별 상이) | 표준 스펙으로 이식성 높음 |
| 역할 분리 | 불가 | 인프라팀/앱팀 명확히 분리 |
| 프로토콜 | HTTP/HTTPS만 | HTTP, HTTPS, TCP, UDP, gRPC |
| 가중치 트래픽 | Controller별 구현 | 표준 스펙 (`weight`) |
| 헤더 기반 라우팅 | annotations | 표준 스펙 |
| GA 상태 | 오래됨 (기능 고정) | v1 GA (2023, HTTPRoute) |

---

## 2. 핵심 3 오브젝트

```
[GatewayClass] ← 인프라팀이 정의 (어떤 구현체를 쓸지)
     ↓
[Gateway]      ← 인프라팀이 생성 (진입점: 포트, 프로토콜, TLS)
     ↓
[HTTPRoute]    ← 앱팀이 정의 (URL → Service 라우팅 규칙)
```

### GatewayClass — 구현체 등록

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
  # 구현체 예시: Nginx Gateway Fabric, Envoy Gateway, Istio 등
```

### Gateway — 진입점 정의 (인프라팀)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: infra
spec:
  gatewayClassName: nginx-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-tls-secret
      # 어떤 namespace의 HTTPRoute를 허용할지
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              kubernetes.io/metadata.name: production
```

### HTTPRoute — 라우팅 규칙 (앱팀)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: order-route
  namespace: production
spec:
  parentRefs:
    - name: my-gateway
      namespace: infra    # 위에서 만든 Gateway 참조

  hostnames:
    - "api.example.com"

  rules:
    # 경로 기반 라우팅
    - matches:
        - path:
            type: PathPrefix
            value: /api/orders
      backendRefs:
        - name: order-svc
          port: 80

    # 헤더 기반 라우팅 (Canary용)
    - matches:
        - headers:
            - name: X-Canary
              value: "true"
      backendRefs:
        - name: order-svc-canary
          port: 80

    # 가중치 기반 트래픽 분할 (Blue-Green/Canary)
    - matches:
        - path:
            type: PathPrefix
            value: /api/users
      backendRefs:
        - name: user-svc-stable
          port: 80
          weight: 90
        - name: user-svc-canary
          port: 80
          weight: 10
```

---

## 3. 역할 분리 모델

```
인프라팀 (cluster-admin)
  → GatewayClass 설치
  → Gateway 생성 (포트, TLS, namespace 허용 범위 결정)

앱팀 (namespace admin)
  → HTTPRoute 생성 (자기 namespace 내 라우팅 규칙만)
  → 인프라팀 승인 없이도 라우팅 변경 가능 (권한 범위 내)
```

→ 기존 Ingress는 모든 설정이 단일 오브젝트에 혼재 → 대규모 팀에서 충돌 발생

---

## 4. 주요 구현체

| 구현체 | 특징 |
|---|---|
| **Nginx Gateway Fabric** | Nginx 공식 Gateway API 구현 |
| **Envoy Gateway** | Envoy 기반, CNCF 프로젝트 |
| **Istio** | Service Mesh + Gateway API 통합 |
| **Traefik** | 가볍고 설정 간단 |
| **Cilium** | eBPF 기반 고성능 |

---

## 5. 마이그레이션 전략

기존 Ingress에서 Gateway API로 전환 시:
1. Gateway API CRD 설치 및 Controller 배포
2. 기존 Ingress와 병행 운영 (트래픽 없이 HTTPRoute 먼저 작성)
3. DNS 또는 LB를 Gateway 쪽으로 전환
4. 기존 Ingress 제거

---

## 6. 관련
- [[Ingress]] · [[Service]] · [[Deployment]]
