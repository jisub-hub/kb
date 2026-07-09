---
tags:
  - k8s
  - configmap
  - secret
  - configuration
created: 2026-06-15
---

# ConfigMap & Secret

> [!summary] 한 줄 요약
> **ConfigMap**: 비민감 설정값(환경변수, 설정파일)을 이미지와 분리. **Secret**: 비밀번호·API Key 등 민감 정보를 Base64 인코딩해 저장. 둘 다 Pod에 환경변수 또는 파일로 주입한다.

---

## 1. ConfigMap

### 생성

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # 키-값 형식
  APP_ENV: "production"
  LOG_LEVEL: "INFO"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"

  # 파일 형식 (application.yaml 통째로)
  application.yaml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:postgresql://postgres-svc:5432/mydb
```

### Pod에서 사용 — 환경변수

```yaml
spec:
  containers:
    - name: myapp
      # 전체 ConfigMap을 환경변수로
      envFrom:
        - configMapRef:
            name: app-config

      # 특정 키만 선택
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

### Pod에서 사용 — 파일 마운트

```yaml
spec:
  volumes:
    - name: config-volume
      configMap:
        name: app-config
  containers:
    - name: myapp
      volumeMounts:
        - name: config-volume
          mountPath: /config
          # /config/application.yaml 로 마운트됨
```

---

## 2. Secret

### 생성

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                      # 평문으로 작성 → k8s가 Base64 자동 인코딩
  DB_USERNAME: "myapp"
  DB_PASSWORD: "super-secret-pw"
  JWT_SECRET: "jwt-signing-key"
```

또는 CLI로:
```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USERNAME=myapp \
  --from-literal=DB_PASSWORD=super-secret-pw
```

### Secret 타입

| 타입 | 용도 |
|---|---|
| `Opaque` | 임의 키-값 (기본값) |
| `kubernetes.io/tls` | TLS 인증서 (`tls.crt`, `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Docker Registry 인증 |
| `kubernetes.io/service-account-token` | ServiceAccount 토큰 |

### Pod에서 사용

```yaml
spec:
  containers:
    - name: myapp
      envFrom:
        - secretRef:
            name: db-secret

      # 특정 키만
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
```

---

## 3. ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| 용도 | 비민감 설정 | 민감 정보 |
| 저장 방식 | 평문 | Base64 인코딩 (etcd 암호화 설정 권장) |
| etcd 암호화 | 기본 없음 | 옵션으로 활성화 가능 |
| RBAC 접근 제어 | 가능 | 가능 (더 엄격히) |
| 노출 위험 | 낮음 | 높음 → 관리 주의 |

> [!warning] Secret은 Base64 인코딩이지 암호화가 아니다
> `echo "super-secret-pw" | base64` 로 누구나 복호화 가능. etcd 암호화 + RBAC + 감사 로그를 함께 적용해야 진짜 보안.

---

## 4. 외부 Secret 관리 도구

k8s 기본 Secret은 보안 한계 → 프로덕션에서는 외부 도구 사용 권장.

| 도구 | 방식 |
|---|---|
| **AWS Secrets Manager** + External Secrets Operator | 클라우드 SM과 k8s Secret 동기화 |
| **HashiCorp Vault** | 전용 비밀 관리 서버, 동적 시크릿 생성 |
| **Sealed Secrets** (Bitnami) | 암호화된 SealedSecret을 Git에 커밋 가능 |
| **OCI Vault** | OCI 환경에서 Vault 서비스 연동 |

### Sealed Secrets 워크플로우

```bash
# 평문 Secret → SealedSecret 암호화 (공개키로)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# SealedSecret을 Git에 커밋 (안전)
git add sealed-secret.yaml

# 클러스터에 적용 → Sealed Secrets Controller가 복호화해 실제 Secret 생성
kubectl apply -f sealed-secret.yaml
```

---

## 5. 변경 반영

- 환경변수로 주입한 경우 → **Pod 재시작 필요** (환경변수는 시작 시 주입)
- 파일 마운트한 경우 → **자동 반영** (기본 약 60초 주기로 갱신)

---

## 6. 관련
- [[Pod]] · [[Deployment]] · [[GitOps]]
