---
tags:
  - deploy
  - kubernetes
  - k8s
  - orchestration
created: 2026-06-15
---

# Kubernetes (K8s)

> [!summary] 한 줄 요약
> 컨테이너 **오케스트레이션** 플랫폼. 배포·확장·자가치유·로드밸런싱·롤링업데이트를 선언적(YAML)으로 자동화한다. [[MSA]] 운영의 사실상 표준.

---

## 1. 핵심 오브젝트
| 오브젝트 | 역할 |
|----------|------|
| **Pod** | 컨테이너 실행 최소 단위(1+ 컨테이너) |
| **Deployment** | Pod 복제·롤링업데이트·롤백 관리 |
| **Service** | Pod 묶음에 안정적 네트워크 엔드포인트(로드밸런싱) |
| **Ingress** | 외부 HTTP 라우팅(도메인/경로 → Service) |
| **ConfigMap / Secret** | 설정 / 민감정보 주입 |
| **HPA** | CPU/메트릭 기반 자동 스케일링 |
| **Namespace** | 논리적 격리 |

```
Internet → Ingress → Service → [Pod, Pod, Pod]  (Deployment가 관리)
```

## 2. Deployment + Service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: order-service }
spec:
  replicas: 3
  selector: { matchLabels: { app: order } }
  template:
    metadata: { labels: { app: order } }
    spec:
      containers:
        - name: order
          image: myorg/order:1.0
          ports: [{ containerPort: 8080 }]
          envFrom:
            - configMapRef: { name: order-config }
            - secretRef: { name: order-secret }
          resources:                       # 요청/제한 (스케줄링·안정성)
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "1Gi" }
          readinessProbe:                  # 트래픽 받을 준비 됐나
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 10
          livenessProbe:                   # 죽었나(재시작 트리거)
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata: { name: order-service }
spec:
  selector: { app: order }
  ports: [{ port: 80, targetPort: 8080 }]
  type: ClusterIP
```
> Spring Boot Actuator의 **liveness/readiness** probe 그룹과 직접 연동된다.

## 3. 자동 스케일링 (HPA)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: order-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: order-service }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

## 4. 설정/시크릿 주입
```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: order-config }
data:
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres:5432/mydb"
```
> Spring은 환경변수를 프로퍼티로 자동 바인딩(`SPRING_DATASOURCE_URL` → `spring.datasource.url`).

## 5. 배포 전략
- **RollingUpdate**(기본): 무중단 점진 교체.
- **Blue/Green, Canary**: 트래픽 분할 검증(Argo Rollouts/Istio).
- **롤백**: `kubectl rollout undo deployment/order-service`.

## 6. 주요 명령
```bash
kubectl apply -f deploy.yaml
kubectl get pods -w
kubectl logs -f deploy/order-service
kubectl rollout status deploy/order-service
kubectl scale deploy/order-service --replicas=5
```

## 7. 생태계
- **Helm**: 패키지/템플릿 관리(values.yaml).
- **Kustomize**: 환경별 오버레이.
- **Service Mesh**(Istio/Linkerd): mTLS·트래픽관리·관측성.
- **GitOps**(ArgoCD/Flux): Git을 단일 진실원으로 자동 동기화.

## 8. 베스트 프랙티스
- resources requests/limits 필수(스케줄링/안정성).
- liveness/readiness probe 정확히(잘못 설정 시 무한 재시작/트래픽 유실).
- 무상태 설계 + 외부 상태(DB/[[Redis]]).
- 관측성 필수: [[Metrics]]·[[Logging]]·[[Tracing]].
- 시크릿은 Secret/외부 볼트(예: External Secrets, Vault).

## 9. 관련
- [[Docker]] · [[CICD]] · [[MSA]] · [[Metrics]] · [[Spring-Cloud-Gateway]]
