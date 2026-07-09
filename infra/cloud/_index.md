---
tags:
  - infra
  - cloud
  - aws
  - serverless
  - moc
  - index
created: 2026-06-16
---

# Cloud MOC

> 클라우드 서비스 활용 패턴 — Serverless, Object Storage, 관리형 서비스.

## 파일 구성
- [[Serverless-Lambda]] — Serverless 유래, Lambda 장단점·제약(15분), API Gateway, EventBridge 스케줄
- [[Object-Storage]] — S3 개념, Presigned URL 업로드·다운로드, 서버 오버헤드 제거, MinIO 로컬 개발
- [[Cost-Optimization]] — 비용 최적화·Right-Sizing: Reserved/Spot, K8s requests/limits, egress, FinOps, GPU 비용

## 관련
- [[../container/Docker-Network-Volume]] — 컨테이너 vs 서버리스 배포 비교
- [[../data-pipeline/Apache-Airflow]] — Lambda 15분 한계 → Airflow/Fargate로 장시간 배치
