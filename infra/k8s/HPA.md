---
tags:
  - k8s
  - hpa
  - autoscaling
  - keda
created: 2026-06-15
---

# HPA — HorizontalPodAutoscaler

> [!summary] 한 줄 요약
> **CPU/메모리 사용량 또는 커스텀 메트릭에 따라 Pod 수를 자동으로 늘리고 줄인다**. KEDA로 Kafka 큐 길이, HTTP 요청 수 등 외부 메트릭 기반 스케일링도 가능.

> [공식 문서](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 참고

---

## 1. HPA 동작 원리

```
[Metrics Server] → HPA Controller가 주기적으로 메트릭 조회
                      ↓ (기본 15초 주기)
                   현재 사용률 vs 목표 사용률 비교
                      ↓
                   Deployment의 replicas 조정
```

### 스케일 계산 공식

```
목표 replicas = ceil(현재 replicas × (현재 메트릭 / 목표 메트릭))

예) replicas=3, 현재 CPU=90%, 목표=60%
  → ceil(3 × 90/60) = ceil(4.5) = 5
```

---

## 2. YAML

### CPU 기반 (기본)

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
  minReplicas: 2          # 최소 replica (0으로 내리면 트래픽 없을 때 완전 종료)
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # 평균 CPU 60% 목표
```

### 메모리 + CPU 복합

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi         # 평균 메모리 400Mi 목표
```

### 커스텀 메트릭 (Prometheus 연동)

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"         # Pod당 초당 100 요청 목표
```

---

## 3. Metrics Server 설치 (필수)

```bash
# HPA는 Metrics Server 없이 동작하지 않음
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 설치 확인
kubectl top pods
kubectl top nodes
```

---

## 4. VPA (VerticalPodAutoscaler) — 수직 스케일

HPA(수평: Pod 수 조정)와 다르게 **CPU/Memory requests/limits를 자동 조정**.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"       # Off / Initial / Recreate / Auto
```

| 모드 | 동작 |
|---|---|
| `Off` | 권장값만 계산, 적용 안 함 (관찰용) |
| `Initial` | Pod 생성 시에만 적용 |
| `Recreate` | 변경 필요 시 Pod 재생성 |
| `Auto` | 자동 적용 (현재 = Recreate) |

> [!warning] HPA + VPA 동시 사용 주의
> CPU/Memory 기반 HPA와 VPA를 같이 쓰면 충돌 가능. VPA는 커스텀 메트릭 기반 HPA와는 함께 사용 가능.

---

## 5. KEDA — 이벤트 기반 스케일링

**Kubernetes Event-Driven Autoscaling**. Kafka 큐, RabbitMQ, Redis, HTTP 요청 수 등 HPA가 지원하지 않는 **외부 메트릭** 기반 스케일링.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

### ScaledObject — Kafka 컨슈머 오토스케일 예시

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: kafka-consumer-deployment
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-group
        topic: my-topic
        lagThreshold: "50"      # 파티션당 미처리 메시지 50개 초과 시 스케일 업
```

### KEDA Scaler 예시

| Scaler | 트리거 메트릭 |
|---|---|
| `kafka` | Consumer lag |
| `rabbitmq` | Queue depth |
| `redis` | List length |
| `prometheus` | PromQL 쿼리 결과 |
| `http` | 활성 HTTP 요청 수 |
| `aws-sqs-queue` | SQS 큐 메시지 수 |

---

## 6. HPA 스케일 동작 제어

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # 급격한 스케일업 방지 (60초 안정화)
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60             # 60초마다 최대 2개씩만 증가
    scaleDown:
      stabilizationWindowSeconds: 300   # 스케일다운은 5분 관찰 후 (안정성)
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60             # 60초마다 최대 10%씩만 감소
```

---

## 7. 유용한 명령어

```bash
# HPA 현재 상태
kubectl get hpa
kubectl describe hpa myapp-hpa

# 실시간 스케일링 모니터링
kubectl get hpa --watch

# 수동 스케일 (테스트용, HPA 있으면 곧 override됨)
kubectl scale deployment myapp --replicas=5
```

---

## 8. 관련
- [[Deployment]] · [[Pod]] · [[Taint-Toleration]]
- [[Resource-Management]] — CPU requests/limits·throttling이 CPU 기준 오토스케일에 주는 영향
