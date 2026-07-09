---
tags:
  - infra
  - ssh
  - cicd
  - moc
  - index
created: 2026-06-15
---

# SSH & CI/CD MOC

> 서버 접속부터 자동화 배포까지 — 개발자가 매일 쓰는 인프라 기초.

## 파일 구성
- [[SSH-SCP]] — SSH 프로토콜, PEM Key 방식, SCP 파일 전송, authorized_keys 등록
- [[GitLab-CICD]] — CI/CD 개념, GitLab 파이프라인 구조, 백엔드/프론트엔드 파이프라인 예시

## 전체 흐름

```
개발자  →  git push  →  GitLab
                           └→ Pipeline 자동 트리거
                                 ├→ Build Stage (Docker 이미지 빌드)
                                 └→ Deploy Stage
                                       └→ SSH로 서버 접속
                                             └→ docker compose up -d
```

## 관련
- [[../network/_index|Network]] · [[../container/Container]] · [[../k8s/Kubernetes]]
