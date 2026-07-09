---
tags:
  - k8s
  - pod
  - workload
created: 2026-06-15
---

# Pod

> [!summary] 한 줄 요약
> Kubernetes **최소 배포 단위**. 하나 이상의 컨테이너가 같은 네트워크·스토리지 네임스페이스를 공유하며 항상 함께 스케줄된다.

---

## 1. Pod란

- 컨테이너 1개 이상을 묶은 논리적 단위
- 같은 Pod 안 컨테이너끼리는 `localhost`로 통신 가능 (같은 네트워크 네임스페이스)
- **임시적(ephemeral)**: Pod는 언제든 삭제·재생성될 수 있다 → 직접 Pod를 생성하지 말고 [[Deployment]]를 사용

---

## 2. Pod YAML 전체 구조

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: default
  labels:
    app: myapp
    version: "1.0"
spec:
  containers:
    - name: myapp
      image: myorg/myapp:1.2.3      # latest 태그 사용 금지
      ports:
        - containerPort: 8080
      
      # 환경변수
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      envFrom:
        - configMapRef:
            name: app-config

      # 리소스 요청/제한 (필수)
      resources:
        requests:
          cpu: "250m"       # 0.25 core 보장 요청
          memory: "256Mi"
        limits:
          cpu: "500m"       # 최대 0.5 core
          memory: "512Mi"   # 초과 시 OOMKilled

      # Probe (아래 섹션 참고)
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
        failureThreshold: 3
      startupProbe:
        httpGet:
          path: /actuator/health
          port: 8080
        failureThreshold: 30   # 30 × 10s = 최대 5분 대기
        periodSeconds: 10

      # 볼륨 마운트
      volumeMounts:
        - name: app-logs
          mountPath: /var/log/myapp
  
  volumes:
    - name: app-logs
      emptyDir: {}             # Pod 생애주기와 함께
  
  # 재시작 정책
  restartPolicy: Always        # Always | OnFailure | Never
```

---

## 3. Probe (상태 체크)

| Probe | 역할 | 실패 시 |
|---|---|---|
| **startupProbe** | 앱이 처음 시작됐는지 체크 (느린 앱용) | Pod 재시작 |
| **livenessProbe** | 앱이 살아있는지 주기적 체크 | Pod 재시작 |
| **readinessProbe** | 트래픽 받을 준비가 됐는지 체크 | Service 엔드포인트에서 제거 (재시작 X) |

**체크 방식 3가지:**
```yaml
# HTTP (가장 일반적)
httpGet:
  path: /health
  port: 8080

# TCP 포트 열림 여부
tcpSocket:
  port: 8080

# 컨테이너 내 명령 실행 (exit 0 = 정상)
exec:
  command: ["cat", "/tmp/healthy"]
```

> [!tip] startupProbe를 쓰는 이유
> livenessProbe만 쓰면 JVM 기동 시간(10~30초)에 실패 판정 → 재시작 루프. startupProbe가 성공하면 그 이후부터 livenessProbe 시작.

---

## 4. 리소스 requests / limits

```
requests: 노드에서 보장 예약하는 최소 자원 (스케줄링 기준)
limits:   컨테이너가 사용할 수 있는 최대 자원
```

| 상황 | 결과 |
|---|---|
| CPU 제한 초과 | **Throttling** (느려짐, 종료 X) |
| Memory 제한 초과 | **OOMKilled** (즉시 종료) |
| requests 미설정 | 스케줄러가 노드 부하 예측 불가 → 노드 과부하 위험 |

**CPU 단위:** `1` = 1 vCPU core, `250m` = 0.25 core  
**Memory 단위:** `256Mi` = 256 MiB, `1Gi` = 1 GiB

---

## 5. Pod Lifecycle

```
Pending → Running → Succeeded / Failed
            ↑            ↓
          (재시작)   CrashLoopBackOff (계속 실패)
```

| 상태 | 설명 |
|---|---|
| `Pending` | 스케줄 대기 또는 이미지 다운로드 중 |
| `Running` | 컨테이너 1개 이상 실행 중 |
| `Succeeded` | 모든 컨테이너 exit 0 (Job/CronJob) |
| `Failed` | 컨테이너가 비정상 종료 |
| `CrashLoopBackOff` | 반복 재시작 실패 → 지수 백오프로 대기 시간 증가 |
| `OOMKilled` | 메모리 제한 초과로 강제 종료 |

---

## 6. 멀티 컨테이너 패턴

```yaml
spec:
  containers:
    - name: main-app
      image: myapp:1.0
    - name: log-sidecar          # Sidecar 패턴
      image: fluentd:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log
```

| 패턴 | 설명 |
|---|---|
| **Sidecar** | 메인 앱 옆에서 로깅·프록시·설정 갱신 보조 |
| **Ambassador** | 외부 서비스로 프록시 (Redis 연결 풀 등) |
| **Adapter** | 메인 앱 출력을 표준 형식으로 변환 |

---

## 7. 관련
- [[ReplicaSet]] · [[Deployment]] · [[ConfigMap-Secret]] · [[PV-PVC]]
