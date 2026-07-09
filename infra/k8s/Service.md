---
tags:
  - k8s
  - service
  - networking
created: 2026-06-15
---

# Service

> [!summary] 한 줄 요약
> **Pod 집합에 안정적인 네트워크 엔드포인트를 제공**하는 오브젝트. Pod는 재생성될 때마다 IP가 바뀌지만 Service IP(ClusterIP)는 고정 → 서비스 간 통신·외부 노출에 필수.

---

## 1. 왜 Service가 필요한가

```
Pod A (10.244.1.5) → 직접 접근 → Pod B (10.244.2.8)
                                    ↑ 재시작 후 IP 변경됨 → 접근 불가
```

Service가 있으면:
```
Pod A → Service (10.96.0.10 고정) → Pod B (IP 변경돼도 무관)
                    ↑ selector로 실시간 Pod 목록 추적
```

---

## 2. Service 타입 4가지

### ClusterIP (기본값) — 클러스터 내부 통신

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
    - port: 80          # Service 포트 (클라이언트가 접근)
      targetPort: 8080  # Pod 컨테이너 포트
  type: ClusterIP       # 생략해도 기본값
```

- 클러스터 내부에서만 접근 가능 (`클러스터IP:80`)
- 서비스명으로도 접근 가능: `myapp-svc.default.svc.cluster.local`
- 용도: 마이크로서비스 간 내부 통신

---

### NodePort — 노드 IP + 포트로 외부 접근

```yaml
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # 30000-32767 범위, 생략 시 자동 할당
  type: NodePort
```

```
외부 → 노드IP:30080 → Service:80 → Pod:8080
```

- 모든 노드의 30080 포트로 접근 가능
- 용도: 개발/테스트 환경. 프로덕션엔 보안·관리 이유로 부적합

---

### LoadBalancer — 클라우드 LB 자동 생성

```yaml
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

- 클라우드 환경(OCI/AWS/GCP)에서 **외부 로드밸런서를 자동 생성**해 공인 IP 할당
- NodePort + ClusterIP를 포함 (상위 집합)
- 용도: 프로덕션 외부 노출. 서비스마다 LB가 생성되므로 비용 발생 → 많아지면 [[Ingress]] 사용

---

### Headless Service — 개별 Pod IP 직접 접근

```yaml
spec:
  selector:
    app: mydb
  clusterIP: None         # Headless
  ports:
    - port: 5432
```

- ClusterIP 없음 → DNS 조회 시 개별 Pod IP 목록 반환
- StatefulSet의 각 Pod에 직접 접근할 때 사용 (예: Kafka broker, Elasticsearch node)
- `mydb-0.mydb-svc.default.svc.cluster.local` 형태로 각 Pod에 직접 접근 가능

---

## 3. 서비스 디스커버리 — DNS

```
서비스명: myapp-svc
네임스페이스: default
→ DNS: myapp-svc.default.svc.cluster.local
→ 같은 네임스페이스 내에서는 myapp-svc 만으로도 접근 가능
```

```yaml
# 같은 namespace에서 연결 예시
env:
  - name: ORDER_SERVICE_URL
    value: "http://order-svc/api/orders"
```

---

## 4. Endpoints

Service와 실제 Pod 사이의 연결 정보.

```bash
kubectl get endpoints myapp-svc
# NAME        ENDPOINTS                       AGE
# myapp-svc   10.244.1.5:8080,10.244.2.8:8080   5m
```

- `selector`와 매칭되는 Ready 상태 Pod의 IP:Port 목록
- readinessProbe 실패 Pod는 자동 제거

---

## 5. 타입 비교 요약

| 타입 | 접근 범위 | 외부 IP | 비용 | 용도 |
|---|---|---|---|---|
| `ClusterIP` | 내부만 | ❌ | 없음 | MSA 내부 통신 |
| `NodePort` | 노드 IP | ❌ (노드 IP 사용) | 없음 | 개발/테스트 |
| `LoadBalancer` | 외부 | ✅ 공인 IP | 클라우드 LB 비용 | 프로덕션 소수 서비스 |
| `Headless` | 내부 직접 | ❌ | 없음 | StatefulSet Pod 직접 접근 |

---

## 6. 관련
- [[Pod]] · [[Deployment]] · [[Ingress]] · [[GatewayAPI]]
