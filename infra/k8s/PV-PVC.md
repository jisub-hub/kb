---
tags:
  - k8s
  - storage
  - pv
  - pvc
  - statefulset
created: 2026-06-15
---

# PersistentVolume & PersistentVolumeClaim

> [!summary] 한 줄 요약
> **PV**: 클러스터에 미리 준비된 스토리지 자원. **PVC**: Pod가 스토리지를 요청하는 방법. PVC를 StorageClass와 함께 쓰면 PV를 자동 생성(동적 프로비저닝)할 수 있다.

> [공식 문서](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 참고

---

## 1. 왜 필요한가

Pod는 임시적 → Pod 삭제 시 `emptyDir`로 마운트한 데이터도 사라짐.
DB, 파일 업로드 등 **데이터를 Pod 수명과 독립적으로 유지**해야 할 때 PV/PVC 사용.

---

## 2. 핵심 개념

```
[클러스터 관리자] → PV 생성 (실제 스토리지 연결)
[개발자]         → PVC 생성 (용량·accessMode 요청)
[k8s]            → PVC ↔ PV 바인딩 (조건 매칭)
[Pod]            → PVC를 볼륨으로 마운트
```

또는 **StorageClass + 동적 프로비저닝**:
```
[개발자]         → PVC 생성 (StorageClass 지정)
[k8s]            → StorageClass Provisioner가 PV 자동 생성 + 바인딩
[Pod]            → PVC를 볼륨으로 마운트
```

---

## 3. PV (PersistentVolume)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce          # 한 노드에서만 읽기/쓰기
  persistentVolumeReclaimPolicy: Retain   # PVC 삭제 후 PV 처리 방식
  storageClassName: standard
  # 실제 스토리지 연결 (예: NFS)
  nfs:
    server: 192.168.1.100
    path: /exports/myapp
```

### Access Mode

| 모드 | 설명 | 사용 사례 |
|---|---|---|
| `ReadWriteOnce` (RWO) | 단일 노드에서 읽기/쓰기 | DB, 일반 앱 |
| `ReadOnlyMany` (ROX) | 여러 노드에서 읽기 전용 | 공유 설정 파일 |
| `ReadWriteMany` (RWX) | 여러 노드에서 읽기/쓰기 | 공유 파일시스템 (NFS) |
| `ReadWriteOncePod` | 단일 **Pod**에서만 (k8s 1.22+) | 강한 단독 접근 보장 |

### Reclaim Policy

| 정책 | PVC 삭제 후 PV 처리 |
|---|---|
| `Retain` | PV 유지 (수동 삭제·재사용) → 프로덕션 데이터 보호 권장 |
| `Delete` | PV와 실제 스토리지 자동 삭제 → 동적 프로비저닝 기본 |
| `Recycle` | 내용 삭제 후 재사용 (deprecated) |

---

## 4. PVC (PersistentVolumeClaim)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard    # 동적 프로비저닝 시 사용할 StorageClass
```

### Pod에서 PVC 마운트

```yaml
spec:
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: my-pvc
  containers:
    - name: myapp
      volumeMounts:
        - name: data-volume
          mountPath: /data
```

---

## 5. StorageClass — 동적 프로비저닝

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs      # 클라우드 프로비저너
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true               # PVC 용량 확장 허용
```

| 클라우드 | Provisioner | 스토리지 |
|---|---|---|
| AWS | `kubernetes.io/aws-ebs` | EBS gp3 |
| OCI | `blockvolume.csi.oraclecloud.com` | Block Volume |
| GCP | `kubernetes.io/gce-pd` | Persistent Disk |
| Azure | `kubernetes.io/azure-disk` | Managed Disk |

---

## 6. StatefulSet과 PVC

StatefulSet은 각 Pod마다 **독립적인 PVC를 자동 생성**한다.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 2
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  # 각 Pod마다 PVC 자동 생성
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi
# 결과: postgres-0 → data-postgres-0 PVC
#       postgres-1 → data-postgres-1 PVC
```

---

## 7. PV 수명주기

```
Available → Bound → Released → Available (Recycle) / Deleted
                      ↓
                  Failed (Reclaim 오류)
```

```bash
# PV/PVC 상태 확인
kubectl get pv
kubectl get pvc

# PVC 상세 (바인딩된 PV 확인)
kubectl describe pvc my-pvc

# PV 강제 삭제 (Released 상태에서 재사용 시)
kubectl patch pv my-pv -p '{"spec":{"claimRef": null}}'
```

---

## 8. 관련
- [[Pod]] · [[Deployment]] · [[ConfigMap-Secret]]
