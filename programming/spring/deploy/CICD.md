---
tags:
  - deploy
  - cicd
  - automation
  - devops
created: 2026-06-15
---

# CI/CD

> [!summary] 한 줄 요약
> **CI(지속적 통합)**: 코드 변경을 자주 병합하고 자동 빌드·테스트. **CD(지속적 배포/전달)**: 검증된 산출물을 자동으로 배포. 빠르고 안전한 릴리즈의 기반.

---

## 1. 파이프라인 단계
```
커밋/PR → [빌드] → [단위/통합 테스트] → [정적분석/보안스캔]
       → [Docker 이미지 빌드 & 푸시] → [스테이징 배포] → [E2E]
       → [프로덕션 배포(승인)] → [모니터링]
```

## 2. GitHub Actions 예시 (Spring + Docker + K8s)
```yaml
name: ci-cd
on:
  push: { branches: [main] }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '21', cache: gradle }
      - run: ./gradlew build           # 컴파일 + 테스트
      - name: Build & push image
        run: |
          ./gradlew bootBuildImage --imageName=ghcr.io/org/order:${{ github.sha }}
          echo ${{ secrets.GHCR_TOKEN }} | docker login ghcr.io -u org --password-stdin
          docker push ghcr.io/org/order:${{ github.sha }}
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: |
          kubectl set image deployment/order-service \
            order=ghcr.io/org/order:${{ github.sha }}   # 롤링 업데이트 트리거
```

## 3. 베스트 프랙티스
- **테스트 자동화**가 CI의 핵심(없으면 무의미).
- 이미지 태그는 **불변**(커밋 SHA), `latest` 지양.
- 시크릿은 CI 시크릿 스토어/볼트로(코드/로그 노출 금지).
- 정적분석/의존성 취약점 스캔(SAST, SCA) 포함.
- **GitOps**(ArgoCD): 매니페스트를 Git에 두고 클러스터가 자동 동기화 → 선언적 CD.
- 배포 후 [[Metrics]]/[[Tracing]] 으로 검증 + 이상 시 자동 롤백.

## 4. 관련
- [[Docker]] · [[Kubernetes]] · [[Metrics]] · [[MSA]]
