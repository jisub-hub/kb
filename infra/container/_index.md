---
tags:
  - infra
  - container
  - docker
  - moc
  - index
created: 2026-06-16
---

# Container MOC

> 컨테이너 기술 — Docker 이미지·네트워크·볼륨, Compose 운영, 무중단 배포 원칙.

## 파일 구성
- [[Container]] — Docker 이미지 빌드, 레이어 캐시, Dockerfile 최적화, Compose 기초
- [[Docker-Network-Volume]] — 네트워크 격리(bridge/host/overlay), 포트 바인딩 보안, Named Volume 백업, Docker Swarm, k3s
- [[Zero-Downtime-Deployment]] — 배포 전략 원칙, Docker Compose/Swarm Rolling, Nginx Blue-Green·Canary, DB 마이그레이션 안전 순서, Spring Graceful Shutdown

## 관련
- [[../k8s/RollingUpdate]] — K8s YAML 배포 전략 (Rolling·Blue-Green·Canary·Argo Rollouts)
- [[../k8s/_index]] — Kubernetes 오케스트레이션
- [[../ssh-cicd/GitLab-CICD]] — 파이프라인에서 Docker 빌드·배포
