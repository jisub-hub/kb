---
tags:
  - k8s
  - replicaset
  - workload
created: 2026-06-15
---

# ReplicaSet

> [!summary] 한 줄 요약
> **지정된 수(replicas)의 Pod가 항상 실행되도록 유지**하는 컨트롤러. 직접 사용하지 않고 [[Deployment]]가 내부적으로 관리한다.

---

## 1. 역할

- Pod가 삭제되거나 노드 장애로 죽으면 **자동으로 새 Pod 생성**해 개수 유지
- `selector.matchLabels` 로 어떤 Pod를 관리할지 결정
- Deployment가 업데이트될 때 새 ReplicaSet 생성, 기존 ReplicaSet 스케일 다운

---

## 2. YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp          # 이 라벨을 가진 Pod를 관리
  template:
    metadata:
      labels:
        app: myapp        # selector와 반드시 일치해야 함
    spec:
      containers:
        - name: myapp
          image: myorg/myapp:1.0
          ports:
            - containerPort: 8080
```

---

## 3. Deployment와의 관계

```
Deployment (업데이트 전략, 롤백 이력 관리)
  ├── ReplicaSet v1 (replicas: 0) ← 이전 버전, 롤백용으로 보존
  └── ReplicaSet v2 (replicas: 3) ← 현재 버전
```

- Deployment가 이미지를 업데이트하면 **새 ReplicaSet**을 만들고 서서히 전환
- 기존 ReplicaSet은 `replicas: 0`으로 보존 → 롤백 시 다시 살림
- `kubectl rollout history deployment/myapp` 으로 RS 버전 이력 확인 가능

---

## 4. 왜 ReplicaSet을 직접 쓰지 않는가

| | ReplicaSet 직접 사용 | Deployment 사용 |
|---|---|---|
| 롤링 업데이트 | ❌ 수동으로 RS 교체 필요 | ✅ 자동 |
| 롤백 | ❌ 직접 RS 조작 | ✅ `kubectl rollout undo` |
| 업데이트 이력 | ❌ 없음 | ✅ revision 이력 보존 |
| Pause/Resume | ❌ | ✅ |

→ **항상 Deployment를 사용.** ReplicaSet은 k8s 내부 구현 디테일로만 이해.

---

## 5. 유용한 명령어

```bash
kubectl get replicasets
kubectl get rs -l app=myapp

# RS가 관리 중인 Pod 확인
kubectl get pods -l app=myapp

# 수동 스케일 (테스트용 — 실제로는 Deployment로)
kubectl scale rs myapp-rs --replicas=5

# RS 상세 (Events 포함)
kubectl describe rs myapp-rs
```

---

## 6. 관련
- [[Pod]] · [[Deployment]] · [[HPA]]
