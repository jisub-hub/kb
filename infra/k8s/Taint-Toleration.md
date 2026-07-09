---
tags:
  - k8s
  - scheduling
  - taint
  - toleration
  - node-affinity
created: 2026-06-15
---

# Taint & Toleration & Node Affinity

> [!summary] 한 줄 요약
> **어떤 Pod가 어떤 노드에 배치될지를 제어**하는 메커니즘. Taint/Toleration은 "이 노드에는 특정 Pod만 허용"이고, Node Affinity는 "이 Pod는 특정 노드에 배치해 달라"는 요청이다.

---

## 1. 배치 제어 메커니즘 비교

| 메커니즘 | 방향 | 강제성 |
|---|---|---|
| **Taint** | 노드 → Pod (거부) | Hard (NoSchedule) / Soft (PreferNoSchedule) |
| **Toleration** | Pod → 노드 (허용) | Taint와 쌍으로 동작 |
| **Node Affinity** | Pod → 노드 (선호/필수) | Hard (required) / Soft (preferred) |
| **Pod Affinity** | Pod → Pod (함께 배치) | Hard / Soft |
| **nodeSelector** | Pod → 노드 라벨 (단순) | Hard |

---

## 2. Taint — 노드에 오염 표시

```bash
# 노드에 Taint 추가
kubectl taint nodes node1 dedicated=gpu:NoSchedule
#                          ↑키=값    ↑효과

# Taint 제거 (-로 끝내기)
kubectl taint nodes node1 dedicated=gpu:NoSchedule-

# 현재 Taint 확인
kubectl describe node node1 | grep Taint
```

### Taint Effect 3가지

| Effect | 의미 |
|---|---|
| `NoSchedule` | Toleration 없는 Pod는 **스케줄 거부** (기존 Pod는 유지) |
| `PreferNoSchedule` | 가능하면 다른 노드에 배치 (Soft) |
| `NoExecute` | Toleration 없는 Pod는 스케줄 거부 + **기존 실행 중인 Pod도 퇴출(evict)** |

---

## 3. Toleration — Pod가 Taint 수용

```yaml
spec:
  tolerations:
    # 특정 Taint를 수용
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

    # 같은 키의 모든 값 수용
    - key: "dedicated"
      operator: "Exists"
      effect: "NoSchedule"

    # NoExecute에 tolerationSeconds: 일정 시간 후 퇴출
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300    # 노드 이상 후 300초까지는 유지
```

### 주요 사용 패턴

```bash
# GPU 노드에 Taint → GPU Pod만 스케줄
kubectl taint nodes gpu-node1 nvidia.com/gpu=true:NoSchedule

# 마스터 노드 Taint (k8s 기본 설정)
# node-role.kubernetes.io/control-plane:NoSchedule
# → 일반 Pod가 마스터 노드에 배치되지 않도록
```

---

## 4. Node Affinity — Pod가 특정 노드 선호/필수 지정

```yaml
spec:
  affinity:
    nodeAffinity:
      # 반드시 이 노드에 배치 (Hard)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values: ["amd64"]
              - key: node-type
                operator: In
                values: ["high-memory"]

      # 가능하면 이 노드 선호 (Soft)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80             # 1~100, 높을수록 강한 선호
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["ap-northeast-2a"]
```

### Operator 종류

| Operator | 의미 |
|---|---|
| `In` | 값 목록 중 하나와 일치 |
| `NotIn` | 값 목록에 없음 |
| `Exists` | 키가 존재 (값 무관) |
| `DoesNotExist` | 키가 없음 |
| `Gt` | 값이 더 큼 (숫자) |
| `Lt` | 값이 더 작음 (숫자) |

---

## 5. Pod Affinity & Anti-Affinity

같은 노드 또는 같은 영역에 Pod를 함께/분산 배치.

```yaml
spec:
  affinity:
    # 같은 zone에 cache Pod가 있는 노드 선호 (함께 배치)
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: cache
            topologyKey: topology.kubernetes.io/zone

    # 같은 노드에 같은 앱 다른 Pod 배치 금지 (분산 배치)
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
          # hostname = 같은 노드에 중복 배치 금지 → HA 구성 필수
```

> [!tip] Anti-Affinity로 HA 구성
> `topologyKey: kubernetes.io/hostname` + Anti-Affinity → 같은 Deployment의 Pod들이 서로 다른 노드에 강제 분산 → 노드 1개 장애 시 나머지 Pod는 다른 노드에서 정상 서비스.

---

## 6. 실제 활용 패턴

| 상황 | 해법 |
|---|---|
| GPU 노드에 ML Pod만 배치 | Taint + Toleration |
| DB Pod를 SSD 노드에만 배치 | Node Affinity (required) |
| Web Pod를 여러 AZ에 분산 | Pod Anti-Affinity (topology: zone) |
| 같은 노드에 중복 배치 방지 | Pod Anti-Affinity (topology: hostname) |
| 특정 고객사 노드 격리 | Taint (NoSchedule) + Toleration |

---

## 7. 관련
- [[Pod]] · [[Deployment]] · [[HPA]]
