---
tags:
  - infra
  - kubernetes
  - k8s
  - kustomize
  - gitops
created: 2026-06-16
---

# Kustomize — 템플릿 없는 Kubernetes YAML 관리

> [!summary] 한 줄 요약
> Kustomize는 원본 YAML을 수정하지 않고 **오버레이(overlay) 방식**으로 환경별 차이만 패치한다. Go 템플릿 없이 순수 YAML만 사용하며 `kubectl -k`로 내장 지원된다.

> [공식 문서](https://kustomize.io/) 참고 | [kubectl 통합 문서](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)

---

## 11.1 Kustomize 개요

### 핵심 철학

- **No Template** — Go 템플릿(`{{ }}`) 없이 순수 YAML 작성
- **오버레이(Overlay)** — base를 복제하지 않고, 환경별 패치(patch)만 따로 관리
- **kubectl 내장** — `kubectl apply -k ./overlays/prod` 로 별도 도구 설치 불필요
- **선언적 GitOps 친화** — ArgoCD / Flux가 kustomization.yaml을 직접 인식

### 설치 (kubectl 내장 외 standalone 사용 시)

```bash
# macOS
brew install kustomize

# 버전 확인
kustomize version
kubectl version --client   # kubectl에 내장된 kustomize 버전도 확인
```

---

## 11.2 base / overlay 디렉토리 구조

```
k8s/
  base/                           # 모든 환경 공통 YAML
    kustomization.yaml
    deployment.yaml
    service.yaml
    hpa.yaml
  overlays/
    dev/
      kustomization.yaml          # base 참조 + dev 전용 패치
      patch-replicas.yaml         # replicas 오버라이드
      patch-resources.yaml        # 리소스 축소
    staging/
      kustomization.yaml
      patch-replicas.yaml
    prod/
      kustomization.yaml          # base 참조 + prod 전용 패치
      patch-resources.yaml        # 리소스 확장
      patch-hpa.yaml              # HPA 설정 강화
```

### base/kustomization.yaml

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml

# 모든 리소스에 공통 레이블 추가
commonLabels:
  app: myapp
  managed-by: kustomize

# 모든 리소스에 공통 어노테이션 추가
commonAnnotations:
  team: backend
```

### base/deployment.yaml (순수 YAML — 템플릿 없음)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp          # namePrefix/nameSuffix로 환경별 이름 자동 추가 가능
spec:
  replicas: 1          # base 기본값 — overlay에서 오버라이드
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myorg/myapp:latest   # images 필드로 태그 교체
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
```

### overlays/prod/kustomization.yaml

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# base 참조 (상대경로)
resources:
  - ../../base

# 이미지 태그 교체 — CI에서 kustomize edit set image 로 자동 업데이트
images:
  - name: myorg/myapp
    newTag: "v1.2.3"        # CI/CD에서 동적으로 교체

# 이름에 -prod 접미사 추가 (선택)
nameSuffix: -prod

# prod 전용 패치 목록
patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
  - path: patch-hpa.yaml

# 공통 레이블 추가 (base 위에 덮어씌움)
commonLabels:
  env: prod
```

---

## 11.3 kustomization.yaml 핵심 필드

### resources — 포함할 YAML/디렉토리 목록

```yaml
resources:
  - deployment.yaml        # 로컬 파일
  - service.yaml
  - ../../base             # 상위 base 디렉토리 참조
  - https://raw.githubusercontent.com/cert-manager/cert-manager/v1.14.0/deploy/crds/  # 원격 URL
```

### patches — 패치 적용

```yaml
patches:
  # 파일로 관리 (Strategic Merge Patch)
  - path: patch-replicas.yaml
    target:
      kind: Deployment
      name: myapp

  # 인라인 패치 (간단한 변경에 적합)
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
      spec:
        replicas: 3
    target:
      kind: Deployment
      name: myapp
```

### images — 이미지 태그 교체

```yaml
images:
  - name: myorg/myapp          # base YAML의 이미지 이름과 일치해야 함
    newName: registry.example.com/myorg/myapp   # 레지스트리 변경 (선택)
    newTag: "abc123def"         # 태그 교체

  - name: redis
    newTag: "7.2.4"
```

### configMapGenerator / secretGenerator

```yaml
# ConfigMap을 파일이나 환경변수 목록에서 자동 생성
configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
      - LOG_LEVEL=INFO
    files:
      - application.properties   # 파일 내용을 ConfigMap에 포함

# Secret 생성 (base64 인코딩 자동)
secretGenerator:
  - name: db-secret
    literals:
      - DB_HOST=postgres.production.svc.cluster.local
      - DB_PORT=5432
    type: Opaque
```

> [!warning] 주의
> configMapGenerator/secretGenerator로 생성된 리소스는 내용 해시가 이름에 추가됨 (예: `app-config-5m8k4t4t7b`). 이로 인해 ConfigMap 변경 시 Deployment 자동 롤링 업데이트 트리거.

### namePrefix / nameSuffix

```yaml
# 모든 리소스 이름 앞/뒤에 문자열 추가
namePrefix: prod-
nameSuffix: -v2

# deployment.yaml의 name: myapp → prod-myapp-v2 로 변환
```

### commonLabels

```yaml
commonLabels:
  app.kubernetes.io/managed-by: kustomize
  app.kubernetes.io/version: "1.2.3"
  env: production
```

---

## 11.4 실전 패치 예시

### Strategic Merge Patch — replicas, resources 변경

```yaml
# overlays/prod/patch-replicas.yaml
# Strategic Merge Patch: metadata.name으로 대상 리소스를 특정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp          # base의 Deployment 이름과 일치해야 함
spec:
  replicas: 5          # base의 replicas: 1 을 5로 오버라이드
```

```yaml
# overlays/prod/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp       # 컨테이너 이름 일치 필요
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

### JSON Patch — 특정 필드 교체/추가/삭제

```yaml
# overlays/dev/patch-disable-hpa.yaml
# JSON Patch: op(operation) + path + value
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  minReplicas: 1
  maxReplicas: 1   # dev에서는 스케일링 비활성화
```

```yaml
# kustomization.yaml에서 JSON Patch 형식으로 지정
patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/imagePullPolicy
        value: Always
      - op: add
        path: /spec/template/metadata/annotations
        value:
          prometheus.io/scrape: "true"
          prometheus.io/port: "8080"
```

### CI/CD에서 이미지 태그 교체

```bash
# kustomize CLI로 이미지 태그 업데이트 (kustomization.yaml 파일 직접 수정)
cd overlays/prod
kustomize edit set image myorg/myapp=myorg/myapp:$CI_COMMIT_SHA

# Git에 커밋 → ArgoCD/Flux가 감지하여 자동 배포 (GitOps)
git add kustomization.yaml
git commit -m "chore: deploy myapp $CI_COMMIT_SHA to prod"
git push

# 또는 kubectl로 직접 적용 (GitOps 미사용 시)
kubectl apply -k overlays/prod/
```

---

## 11.5 Helm vs Kustomize 선택 기준

| 상황 | 권장 도구 |
|------|-----------|
| nginx-ingress, cert-manager 등 오픈소스 앱 설치 | **Helm** |
| 자체 서비스 환경별 배포 관리 (dev/staging/prod) | **Kustomize** |
| 복잡한 조건부 로직 필요 (`if`, `range`) | **Helm** |
| GitOps + ArgoCD / Flux | **Kustomize** (또는 Helm + Helmfile) |
| 팀이 템플릿 언어에 익숙하지 않음 | **Kustomize** |
| 다수 차트를 중앙에서 버전 관리 | **Helm + Helmfile** |
| 간단한 이미지 태그 교체만 필요 | **Kustomize** (`images` 필드) |
| 외부 Chart 커스터마이징 | **Helm** (values.yaml 오버라이드) |

### 혼용 패턴 (Helm + Kustomize)

```bash
# Helm으로 렌더링 후 Kustomize로 후처리
helm template myapp ./mychart -f values-prod.yaml > rendered.yaml
# rendered.yaml을 base로 Kustomize 오버레이 적용
```

```yaml
# ArgoCD에서 Helm과 Kustomize 혼용
# Application.yaml
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: overlays/prod
    kustomize:
      images:
        - myorg/myapp:v1.2.3
```

---

## 11.6 kubectl -k 사용법

```bash
# overlay 디렉토리 지정
kubectl apply -k overlays/prod/
kubectl apply -k overlays/dev/ -n development

# 렌더링 결과만 확인 (dry-run)
kubectl kustomize overlays/prod/          # stdout으로 출력
kustomize build overlays/prod/            # standalone kustomize 사용

# 삭제
kubectl delete -k overlays/prod/

# diff (현재 클러스터 상태와 비교)
kubectl diff -k overlays/prod/
```

---

## 관련
- [[Kubernetes]] — K8s 핵심 개념
- [[Helm]] — Chart 기반 패키지 관리 (Kustomize와 비교)
- [[GitOps]] — ArgoCD + Kustomize 연동 (GitOps 표준 패턴)
- [[Deployment]] · [[ConfigMap-Secret]]
