---
tags:
  - infra
  - kubernetes
  - k8s
  - helm
  - package-manager
created: 2026-06-16
---

# Helm — Kubernetes 패키지 관리자

> [!summary] 한 줄 요약
> Helm은 Kubernetes 애플리케이션을 **Chart**라는 패키지 단위로 묶어 설치·업그레이드·롤백을 선언적으로 관리한다. Go 템플릿 기반으로 환경별 values.yaml로 하나의 Chart를 재사용한다.

> [공식 문서](https://helm.sh/docs/) 참고

---

## 10.1 Helm 개요

Helm은 Kubernetes의 **패키지 매니저** — apt/brew처럼 Chart 저장소에서 앱을 설치하거나, 직접 Chart를 만들어 재사용할 수 있다.

### Helm vs raw YAML vs Kustomize 비교

| 항목 | Raw YAML | Helm | Kustomize |
|------|----------|------|-----------|
| 템플릿 | 없음 | Go 템플릿 | 없음 (오버레이) |
| 패키지 재사용 | 직접 복붙 | Chart 저장소 | base 공유 |
| 환경별 분기 | 파일 복제 | values.yaml | overlays/ |
| 조건부 로직 | 불가 | 가능 (`if`/`range`) | 제한적 |
| 학습 곡선 | 낮음 | 중간 | 낮음 |
| kubectl 내장 | O | X (별도 설치) | O (`kubectl -k`) |
| 주 사용처 | 단순 배포 | 오픈소스 앱 설치 | 자체 서비스 환경 관리 |

---

## 10.2 Chart 구조

```
mychart/
  Chart.yaml          # 차트 메타데이터 (이름, 버전, 설명)
  values.yaml         # 기본 설정값 (사용자가 오버라이드 가능)
  templates/          # Go 템플릿 YAML 파일들
    deployment.yaml
    service.yaml
    ingress.yaml
    _helpers.tpl      # 공통 함수/헬퍼 정의 (이름 앞에 _ 붙이면 렌더링 제외)
  charts/             # 의존성 차트 (서브차트) — helm dependency update로 내려받음
  .helmignore         # 패키징 시 제외할 파일 목록 (.gitignore 형식)
```

### Chart.yaml 예시
```yaml
apiVersion: v2
name: myapp
description: Spring Boot 마이크로서비스 배포용 Chart
type: application        # application | library
version: 0.1.0           # Chart 버전 (semver)
appVersion: "1.0.0"      # 실제 애플리케이션 버전 (정보성)
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled   # values.yaml에서 활성화 여부 제어
```

### values.yaml 예시
```yaml
replicaCount: 2

image:
  repository: myorg/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

ingress:
  enabled: false
  host: ""

postgresql:
  enabled: false    # 서브차트 활성화 여부
```

### templates/deployment.yaml 예시
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}       # _helpers.tpl 함수 호출
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}          # values.yaml 참조
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.springProfile | default "prod" }}
```

---

## 10.3 핵심 명령어

### 설치 / 업그레이드 / 롤백

```bash
# Chart 설치 (릴리스 이름 + Chart 경로/저장소)
helm install myapp ./mychart
helm install myapp ./mychart -n production --create-namespace

# 업그레이드 (없으면 설치, 있으면 업그레이드)
helm upgrade --install myapp ./mychart \
  --set image.tag=v2.0.0 \
  -n production

# 롤백 (이전 버전으로 되돌리기)
helm rollback myapp 1            # revision 1로 롤백
helm history myapp               # 릴리스 히스토리 조회

# 제거
helm uninstall myapp -n production
```

### 저장소 관리

```bash
# 저장소 추가 및 업데이트
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Chart 검색
helm search repo nginx
helm search hub prometheus         # Artifact Hub 검색
```

### 디버깅 / Dry-run

```bash
# 템플릿 렌더링 결과 확인 (실제 배포 없음)
helm template myapp ./mychart -f values-prod.yaml

# dry-run — 서버에 연결해 유효성 검사까지 수행
helm install myapp ./mychart --dry-run --debug

# 현재 배포된 values 확인
helm get values myapp -n production
helm get all myapp -n production    # manifests + values 모두 조회
```

### --set vs -f 비교

```bash
# --set : 커맨드라인에서 단일 값 오버라이드 (CI/CD에 적합)
helm upgrade --install myapp ./mychart \
  --set image.tag=$CI_COMMIT_SHA \
  --set replicaCount=3

# -f : values 파일 전체 오버라이드 (환경별 설정 파일 관리에 적합)
helm upgrade --install myapp ./mychart \
  -f values-prod.yaml

# 두 방식 혼용 가능 — --set이 -f보다 우선순위 높음
helm upgrade --install myapp ./mychart \
  -f values-prod.yaml \
  --set image.tag=$CI_COMMIT_SHA   # CI에서 태그만 동적으로 주입
```

---

## 10.4 values.yaml 오버라이드 패턴

### 환경별 values 파일 분리

```
mychart/
  values.yaml           # 기본값 (모든 환경 공통)
  values-dev.yaml       # dev 환경 오버라이드
  values-staging.yaml   # staging 환경 오버라이드
  values-prod.yaml      # prod 환경 오버라이드
```

```yaml
# values-dev.yaml — dev 환경은 replicaCount 1, 리소스 축소
replicaCount: 1
image:
  tag: "latest"
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
ingress:
  enabled: true
  host: myapp.dev.example.com
```

```yaml
# values-prod.yaml — prod 환경은 고가용성 설정
replicaCount: 3
image:
  tag: ""     # CI에서 --set image.tag=$SHA 로 주입
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
ingress:
  enabled: true
  host: myapp.example.com
```

### Secret 처리

Helm은 values.yaml에 평문 시크릿을 저장하면 Git에 노출되는 문제가 있다.

```bash
# 방법 1: helm-secrets 플러그인 (SOPS + Age/GPG 암호화)
helm plugin install https://github.com/jkroepke/helm-secrets
helm secrets upgrade myapp ./mychart -f secrets.yaml.enc

# 방법 2: External Secrets Operator로 위임 (권장)
# Helm values에는 SecretStore 참조만 두고, 실제 값은 AWS Secrets Manager/Vault에서 주입
# → helm values.yaml에 시크릿 값 자체가 없으므로 Git 노출 위험 없음
```

---

## 10.5 Helmfile — 다중 릴리스 선언형 관리

Helm은 릴리스 하나씩 명령어로 관리 → 여러 서비스를 동시에 관리하기 불편.
**Helmfile**은 여러 Helm 릴리스를 YAML 파일로 선언적으로 관리한다.

```bash
# 설치
brew install helmfile
```

### helmfile.yaml 구조

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx

environments:
  dev:
    values:
      - environments/dev.yaml
  prod:
    values:
      - environments/prod.yaml

releases:
  # 인프라 컴포넌트
  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    version: "4.10.0"
    values:
      - charts/ingress-nginx/values.yaml

  - name: cert-manager
    namespace: cert-manager
    chart: jetstack/cert-manager
    version: "1.14.0"
    set:
      - name: installCRDs
        value: true

  # 애플리케이션 릴리스
  - name: order-service
    namespace: production
    chart: ./charts/order-service
    values:
      - charts/order-service/values.yaml
      - charts/order-service/values-{{ .Environment.Name }}.yaml  # 환경별 values
    set:
      - name: image.tag
        value: {{ requiredEnv "IMAGE_TAG" }}   # 환경변수에서 주입

  - name: user-service
    namespace: production
    chart: ./charts/user-service
    values:
      - charts/user-service/values.yaml
    needs:
      - production/order-service    # 의존 순서 지정
```

### Helmfile 주요 명령어

```bash
# 모든 릴리스 동기화 (없으면 설치, 변경 있으면 업그레이드)
helmfile sync

# 변경 사항 미리 확인 (diff — helm-diff 플러그인 필요)
helm plugin install https://github.com/databus23/helm-diff
helmfile diff

# 특정 릴리스만 적용
helmfile apply -l name=order-service

# 환경 지정
helmfile -e prod sync

# 템플릿 렌더링 결과 확인
helmfile template
```

---

## 10.6 실전 패턴 — Spring Boot 배포

### Chart values.yaml 설계

```yaml
# charts/myapp/values.yaml
image:
  repository: registry.example.com/myorg/myapp
  tag: "latest"          # CI에서 --set image.tag=$SHA 로 교체
  pullPolicy: IfNotPresent

replicaCount: 2

spring:
  profile: prod          # SPRING_PROFILES_ACTIVE 환경변수로 주입

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "1Gi"

# Readiness / Liveness probe 설정
probes:
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 15
    periodSeconds: 10
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 20

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60

externalSecret:
  enabled: true
  secretStoreName: aws-secrets-manager
  secretPath: production/myapp
```

### CI/CD 파이프라인 연동 (GitLab CI 예시)

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  script:
    # Helm 업그레이드 — 없으면 설치, 있으면 업그레이드 (--install 플래그)
    - helm upgrade --install myapp ./charts/myapp
        --namespace production
        --create-namespace
        --set image.tag=$CI_COMMIT_SHA
        --set image.repository=$CI_REGISTRY_IMAGE
        -f ./charts/myapp/values-prod.yaml
        --wait                   # 롤아웃 완료까지 대기
        --timeout 5m
        --atomic                 # 실패 시 자동 롤백
  environment:
    name: production
  only:
    - main
```

---

## 관련
- [[Kubernetes]] — K8s 핵심 개념
- [[Kustomize]] — 템플릿 없는 YAML 오버레이 방식 (Helm과 비교)
- [[GitOps]] — ArgoCD + Helm/Helmfile 연동
- [[ConfigMap-Secret]] · [[RBAC]]
