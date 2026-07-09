---
tags:
  - deploy
  - container
  - moc
  - index
created: 2026-06-15
---

# 🚢 Container & Deploy MOC

> 컨테이너화 & 오케스트레이션 & 배포.

- [[Docker]] — 컨테이너 이미지/빌드/실행
- [[Kubernetes]] — 컨테이너 오케스트레이션(배포·확장·운영)
- [[CICD]] — 빌드/테스트/배포 자동화 파이프라인
- [[Deployment-Strategies]] — Blue-Green/Canary/Rolling, 무중단, Argo Rollouts, expand-contract
- [[Feature-Flags]] — 배포≠출시 분리, 점진 롤아웃·다크런치·킬스위치·A/B, 토글 부채 관리

## 관련
- [[MSA]] · [[../observability/_index|Observability MOC]] · [[Spring-Cloud-Gateway]]

---

## 🧭 흐름
```
소스 → [CI: 빌드·테스트] → [Docker 이미지] → 레지스트리
     → [CD: K8s 배포] → 롤링업데이트 → 모니터링([[Metrics]])
```
