---
tags:
  - k8s
  - security
  - pod
  - securitycontext
created: 2026-06-16
---

# K8s Security Context — Pod / Container 보안 설정

> [!summary] 한 줄 요약
> SecurityContext는 Pod/Container 수준의 **OS 보안 설정**이다. root 실행 방지, 파일시스템 읽기 전용, 권한 상승 차단 등을 선언해 컨테이너 탈출 공격 표면을 줄인다.

---

## 1. SecurityContext 위치

```yaml
spec:
  securityContext:       # ← Pod 수준 (모든 컨테이너에 적용)
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
    - name: app
      securityContext:   # ← Container 수준 (개별 컨테이너에 적용, Pod 설정 오버라이드)
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

---

## 2. 핵심 설정 항목

### Pod 수준

```yaml
spec:
  securityContext:
    runAsNonRoot: true          # root(UID 0)로 실행 시 파드 시작 거부
    runAsUser: 1000             # 특정 UID로 실행 (Dockerfile USER와 일치)
    runAsGroup: 3000            # GID 지정
    fsGroup: 2000               # 볼륨 소유 GID (파일 쓰기 권한)
    seccompProfile:
      type: RuntimeDefault      # 기본 seccomp 프로파일 (syscall 필터링)
    supplementalGroups: [4000]
```

### Container 수준

```yaml
containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false   # sudo/setuid 비트 실행 차단 ⭐
      readOnlyRootFilesystem: true      # 루트 파일시스템 읽기 전용 ⭐
      capabilities:
        drop: ["ALL"]                   # 모든 Linux capabilities 제거 ⭐
        add: ["NET_BIND_SERVICE"]       # 필요한 것만 추가 (1024 미만 포트 바인딩)
      privileged: false                 # 호스트 수준 권한 (절대 true 금지)
      runAsNonRoot: true
      runAsUser: 1000
```

---

## 3. 실전 보안 Pod 템플릿

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: backend-sa
      automountServiceAccountToken: false   # [[RBAC]] 참고
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: backend
          image: myorg/backend:1.2.3         # latest 금지, 정확한 태그
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests: { cpu: "100m", memory: "256Mi" }
            limits:   { cpu: "500m", memory: "512Mi" }
          volumeMounts:
            - name: tmp
              mountPath: /tmp               # readOnlyRootFilesystem이면 /tmp도 별도 마운트
            - name: logs
              mountPath: /app/logs
      volumes:
        - name: tmp
          emptyDir: {}
        - name: logs
          emptyDir: {}
```

---

## 4. Pod Security Admission (PSA)

K8s 1.25+에서 PodSecurityPolicy(PSP)가 제거되고 PSA로 대체됨.

```yaml
# 네임스페이스 라벨로 보안 프로파일 설정
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted   # 위반 파드 차단
    pod-security.kubernetes.io/warn: restricted      # 경고만
    pod-security.kubernetes.io/audit: restricted     # 감사 로그만
```

| 프로파일 | 내용 |
|---------|------|
| `privileged` | 제한 없음 (시스템 컴포넌트용) |
| `baseline` | 최소 제한 (명백한 권한 상승 차단) |
| `restricted` | 최대 제한 (root 금지, caps drop ALL, readOnly 권장) |

```bash
# 현재 NS의 PSA 설정 확인
kubectl get namespace production -o yaml | grep pod-security

# restricted 위반 파드 시뮬레이션 (dry-run)
kubectl label namespace production \
  pod-security.kubernetes.io/warn=restricted --dry-run=server
```

---

## 5. 보안 스캔

```bash
# kube-bench: CIS Kubernetes Benchmark 검사
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# kubesec: Deployment YAML 보안 점수
kubesec scan deployment.yaml

# Trivy: K8s 클러스터 전체 스캔
trivy k8s --report summary cluster
trivy k8s --namespace production --report all cluster
```

---

## 6. 관련
- [[RBAC]] · [[Network-Policy]] · [[Kubernetes]] · [[Pod]]
- [[Web-Attacks]] · [[../security/Cloud-Security-Products]]
