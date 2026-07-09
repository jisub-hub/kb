---
tags:
  - k8s
  - deployment
  - workload
  - rolling-update
created: 2026-06-15
---

# Deployment

> [!summary] 한 줄 요약
> **Pod를 선언적으로 관리하는 핵심 워크로드 오브젝트**. ReplicaSet을 내부적으로 생성·교체하며, 롤링 업데이트·롤백·일시정지를 제공한다. 상태가 없는(Stateless) 애플리케이션의 표준 배포 방식.

---

## 1. 완전한 YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3

  selector:
    matchLabels:
      app: myapp            # 이 라벨의 Pod를 관리

  # 업데이트 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 최대 replicas+1개까지 일시 초과 허용
      maxUnavailable: 0     # 항상 replicas 수 유지 (무중단 필수)

  # 몇 개의 이전 RS를 보존할지 (롤백용)
  revisionHistoryLimit: 5

  template:
    metadata:
      labels:
        app: myapp
    spec:
      # 안전한 종료 대기 시간 (기본 30s)
      terminationGracePeriodSeconds: 60

      containers:
        - name: myapp
          image: myorg/myapp:1.2.3
          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          # 시작 완료 대기 (느린 JVM 앱용)
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          # 트래픽 수신 준비 확인
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5

          # 살아있는지 확인 → 실패 시 재시작
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

## 2. 롤링 업데이트 흐름

```
Before:  [v1] [v1] [v1]   replicas=3

maxSurge=1, maxUnavailable=0 → 한 번에 1개씩 교체:

Step 1:  [v1] [v1] [v1] [v2]   (v2 추가, surge=1)
Step 2:  [v1] [v1] [v2]        (v1 하나 종료, readiness 통과 후)
Step 3:  [v1] [v2] [v2]
Step 4:  [v2] [v2] [v2]        완료
```

- `maxSurge: 1` → 최대 replicas+1개까지 일시적으로 초과 생성
- `maxUnavailable: 0` → 기존 Pod는 새 Pod가 ready 된 후에만 제거 (무중단 핵심)

---

## 3. 이미지 업데이트 & 롤백

```bash
# 이미지 업데이트 (직접 명령어)
kubectl set image deployment/myapp myapp=myorg/myapp:1.3.0

# 업데이트 상태 확인
kubectl rollout status deployment/myapp

# 롤아웃 이력 확인
kubectl rollout history deployment/myapp

# 이전 버전으로 롤백
kubectl rollout undo deployment/myapp

# 특정 revision으로 롤백
kubectl rollout undo deployment/myapp --to-revision=2

# 롤아웃 일시정지 (카나리 배포 시 활용)
kubectl rollout pause deployment/myapp
kubectl rollout resume deployment/myapp
```

> [!tip] 변경 사유 기록
> `kubectl annotate deployment/myapp kubernetes.io/change-cause="image update to 1.3.0"` 로 이력에 사유를 남기면 `rollout history` 에서 확인 가능.

---

## 4. Deployment 적용 전략

| 전략 | 방식 | 다운타임 | 상세 |
|---|---|---|---|
| **RollingUpdate** | 점진적 교체 | 없음 | k8s 기본값, [[RollingUpdate]] 참고 |
| **Recreate** | 전체 종료 후 생성 | 있음 | 버전 혼재 불가 앱 (DB 스키마 등) |

Blue-Green, Canary는 k8s 네이티브 Strategy가 아니며 Service/Ingress 조합 또는 Argo Rollouts로 구현. → [[RollingUpdate]] 참고

---

## 5. Deployment vs StatefulSet vs DaemonSet

| | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| 대상 | Stateless 앱 | 상태 있는 앱 (DB, Kafka) | 모든 노드에 1개씩 |
| Pod 이름 | 랜덤 해시 | 순서 번호 (`web-0`, `web-1`) | 노드당 1개 |
| 스토리지 | 공유 볼륨 | 각자 PVC | — |
| 스케일링 | 임의 순서 | 순차적 (0→1→2) | 노드 수에 따라 |
| 예시 | Spring Boot, Nginx | PostgreSQL, Elasticsearch | log-agent, CNI plugin |

---

## 6. 유용한 명령어

```bash
# 생성/적용
kubectl apply -f deployment.yaml

# 상태 확인
kubectl get deployment myapp
kubectl describe deployment myapp

# 실시간 롤아웃 확인
kubectl rollout status deployment/myapp --watch

# 스케일 조정
kubectl scale deployment myapp --replicas=5

# 현재 이미지 확인
kubectl get deployment myapp -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## 7. 관련
- [[Pod]] · [[ReplicaSet]] · [[RollingUpdate]] · [[HPA]] · [[Service]]
