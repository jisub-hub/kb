---
tags:
  - cicd
  - gitlab
  - pipeline
  - devops
  - automation
created: 2026-06-15
---

# GitLab CI/CD

> [!summary] 한 줄 요약
> `git push` 한 번으로 **빌드 → 테스트 → 배포까지 자동 실행**되는 파이프라인. `.gitlab-ci.yml` 파일 하나로 전체 파이프라인을 선언적으로 정의한다.

---

## 1. 수동 배포 vs CI/CD 자동화

### 수동 배포의 문제
1. 로컬 빌드 → 2. 테스트 (종종 생략) → 3. SCP 전송 → 4. SSH 접속 → 5. 서비스 재시작

| 문제 | 영향 |
|---|---|
| 반복 수작업 | 사람 실수 (잘못된 서버 배포, 파일 누락 등) |
| 팀원마다 절차 다름 | 일관성 없는 배포 |
| 테스트 생략 위험 | 버그가 운영에 바로 반영 |
| 배포 이력 추적 불가 | 장애 시 롤백 기준 불명확 |

### CI/CD 자동화 이후
```
git push origin dev
  → GitLab이 .gitlab-ci.yml 감지
  → Runner가 파이프라인 자동 실행
  → Build → Test → Deploy 순차 실행
  → 실패 시 자동 중단 (잘못된 배포 방지)
```

---

## 2. 핵심 개념

| 개념 | 설명 |
|---|---|
| **Pipeline** | push/MR 이벤트 발생 시 자동 실행되는 전체 작업 흐름 |
| **Stage** | 빌드·테스트·배포 등 순서대로 실행되는 단계 그룹. 같은 Stage 내 Job은 병렬 실행 |
| **Job** | Stage 안의 실제 명령어 실행 단위. Runner가 독립 컨테이너에서 실행 |
| **Runner** | Job을 실제 실행하는 에이전트 (GitLab SaaS Runner 또는 Self-hosted) |
| **Variables** | 비밀키·설정값을 GitLab UI에서 안전하게 주입 (코드에 직접 넣지 않음) |
| **Artifacts** | Job 간 파일 전달 (예: build Job이 만든 `dist/`를 deploy Job이 사용) |
| **Cache** | `node_modules/` 같은 의존성을 재사용해 빌드 속도 향상 |

---

## 3. `.gitlab-ci.yml` 기본 구조

```yaml
# 실행 순서 정의
stages:
  - build
  - test
  - deploy

# 전역 변수
variables:
  IMAGE: "registry.example.com/myapp:latest"

# build Stage의 Job
build-job:
  stage: build
  image: docker:27
  script:
    - docker build -t $IMAGE .
    - docker push $IMAGE
  only:
    - dev          # dev 브랜치에서만 실행

# test Stage의 Job
test-job:
  stage: test
  image: gradle:8-jdk21
  script:
    - ./gradlew test
  needs:
    - build-job    # build-job 완료 후 실행 (Stage 순서 무관)

# deploy Stage의 Job
deploy-job:
  stage: deploy
  image: alpine:latest
  needs:
    - test-job
  script:
    - ssh $SERVER "docker compose pull && docker compose up -d"
```

---

## 4. 백엔드 CI/CD 파이프라인 (Docker 방식)

```yaml
stages:
  - build
  - deploy

variables:
  OCI_REGISTRY: "ap-seoul-1.ocir.io"
  NAMESPACE: "mynamespace"
  REPO: "myapp"
  FULL_IMAGE: "${OCI_REGISTRY}/${NAMESPACE}/${REPO}:${CI_COMMIT_SHORT_SHA}"

# ─── Build Stage ─────────────────────────────
build:
  stage: build
  image: docker:27
  services:
    - docker:27-dind              # Docker-in-Docker
  before_script:
    # OCI Container Registry 로그인
    - echo "$OCIR_TOKEN" | docker login ${OCI_REGISTRY} \
        -u "$OCIR_USER" --password-stdin
    # 멀티 아키텍처 빌드 환경 설정
    - docker buildx create --name multiarch --use
  script:
    - docker buildx build \
        --platform linux/amd64 \
        --cache-from type=registry,ref=${FULL_IMAGE} \
        --tag ${FULL_IMAGE} \
        --push .
  only:
    - dev

# ─── Deploy Stage ────────────────────────────
deploy:
  stage: deploy
  image: alpine:latest
  needs:
    - build
  before_script:
    - apk add --no-cache openssh-client
    # GitLab Variables에서 SSH 개인키 주입
    - mkdir -p ~/.ssh
    - echo "$DEV_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    # 서버 Host Key 미리 등록 (접속 시 yes/no 방지)
    - ssh-keyscan -H ${DEV_HOST} >> ~/.ssh/known_hosts
  script:
    # 서버에서 최신 이미지 pull
    - ssh ${DEV_USER}@${DEV_HOST} "cd ${DEPLOY_PATH} && docker compose pull"
    # 컨테이너 재시작
    - ssh ${DEV_USER}@${DEV_HOST} "docker compose up -d"
    # 미사용 이미지·컨테이너 정리
    - ssh ${DEV_USER}@${DEV_HOST} "docker system prune -f"
  only:
    - dev
```

**GitLab Variables 설정 (Settings > CI/CD > Variables):**

| 변수명 | 내용 | Masked |
|---|---|---|
| `DEV_KEY` | SSH 개인키 전체 내용 | ✅ |
| `DEV_HOST` | 배포 서버 IP | - |
| `DEV_USER` | SSH 접속 사용자명 | - |
| `DEPLOY_PATH` | 서버 배포 경로 | - |
| `OCIR_USER` | Registry 사용자명 | - |
| `OCIR_TOKEN` | Registry 인증 토큰 | ✅ |

---

## 5. 프론트엔드 CI/CD 파이프라인 (npm + rsync)

```yaml
stages:
  - build
  - deploy

# ─── Build Stage ─────────────────────────────
build:
  stage: build
  image: node:20-alpine
  # node_modules 캐싱 (브랜치별)
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  before_script:
    - npm install --legacy-peer-deps
  script:
    - npm run build
  # 빌드 결과물을 다음 stage로 전달
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
  only:
    - dev

# ─── Deploy Stage ────────────────────────────
deploy:
  stage: deploy
  image: alpine:latest
  needs:
    - build        # build artifacts(dist/)를 자동으로 가져옴
  before_script:
    - apk add --no-cache openssh-client rsync
    - mkdir -p ~/.ssh
    - echo "$DEV_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H ${DEV_HOST} >> ~/.ssh/known_hosts
  script:
    # 서버 기존 파일 제거
    - ssh ${DEV_USER}@${DEV_HOST} "rm -rf ${DEPLOY_PATH}/*"
    # rsync로 변경 파일만 효율적 동기화
    - rsync -avz --delete dist/ \
        ${DEV_USER}@${DEV_HOST}:${DEPLOY_PATH}/
    # 파일 권한 설정 (nginx가 읽을 수 있도록)
    - ssh ${DEV_USER}@${DEV_HOST} "chmod -R 755 ${DEPLOY_PATH}"
  only:
    - dev
```

---

## 6. 환경별 파이프라인 분기

```yaml
deploy-dev:
  stage: deploy
  script:
    - echo "Deploy to DEV"
  only:
    - dev              # dev 브랜치에서만

deploy-prod:
  stage: deploy
  script:
    - echo "Deploy to PROD"
  only:
    - main             # main 브랜치에서만
  when: manual         # 프로덕션은 수동 승인 후 실행
```

---

## 7. 파이프라인 최적화 팁

| 항목 | 방법 |
|---|---|
| 빌드 속도 | `cache` 로 의존성 재사용 (`node_modules/`, Gradle 캐시 등) |
| 이미지 빌드 속도 | `--cache-from` 으로 레지스트리 캐시 활용 |
| Stage 병렬화 | 같은 stage의 Job은 자동 병렬 실행 |
| 불필요 실행 방지 | `only`/`except` 또는 `rules` 로 브랜치·조건 제한 |
| 실패 알림 | GitLab Integrations → Slack/Email 연동 |

---

## 8. 관련
- [[SSH-SCP]] · [[../container/Container]] · [[../k8s/Kubernetes]]
