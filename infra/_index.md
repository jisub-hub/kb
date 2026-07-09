---
tags:
  - infra
  - moc
  - index
created: 2026-06-15
---

# Infrastructure MOC

> 애플리케이션 실행 환경을 코드로 정의하고 자동화하는 도구 모음.

## 폴더 구성

### 컴퓨팅 & 오케스트레이션
- [[container/_index|Container]] — Docker 이미지·네트워크·볼륨, Swarm, 무중단 배포
- [[k8s/_index|Kubernetes]] — Pod·Deployment·Service·Ingress·GatewayAPI·PV/PVC·HPA·GitOps·StatefulSet
- [[cloud/_index|Cloud]] — Serverless/Lambda(FaaS), Object Storage·Presigned URL

### 데이터 & 파이프라인
- [[database/_index|Database]] — ACID·CAP·RDB vs NoSQL, PostgreSQL·MongoDB·Elasticsearch·HA 클러스터
- [[data-pipeline/_index|Data Pipeline]] — Apache Airflow DAG·ETL·Scheduler

### 인프라 기반
- [[linux/_index|Linux]] — ulimit, sysctl 커널 파라미터, Ramdisk, TCPDump·Wireshark, LVM
- [[network/_index|Network]] — IP·서브네팅·VCN/VPC·게이트웨이·VPN·LB·보안·HA·대역폭
- [[ssh-cicd/_index|SSH & CI/CD]] — SSH·SCP·공개키 등록·GitLab 파이프라인
- [[architecture/_index|Architecture]] — 3-Tier, K8s 기반, MSA 환경 아키텍처 패턴

### 관측성 & 보안
- [[observability/_index|Observability]] — Prometheus·Grafana·Loki(PLG), ELK Stack, 로그 전략
- [[security/Cloud-Security-Products|Security]] — 클라우드 보안 제품 (WAF·CSPM·SIEM)

### IaC
- [[terraform/Terraform|Terraform]] — 클라우드 리소스 선언적 관리
- [[ansible/Ansible|Ansible]] — 에이전트리스 서버 설정 자동화

## 도구별 역할 구분

| 도구 | 무엇을 관리 | 방식 |
|------|------------|------|
| **Docker** | 애플리케이션 패키징 | 이미지 빌드 |
| **Kubernetes** | 컨테이너 실행·스케일링 | 선언적 오브젝트 |
| **Terraform** | 클라우드 인프라(VPC·RDS·EKS 등) | HCL 선언 |
| **Ansible** | OS 설정·패키지·파일 배포 | YAML Playbook |

## 전형적인 사용 흐름
```
개발자 코드 push
  → CI: 테스트 + Docker 이미지 빌드
  → Terraform: 클라우드 인프라 프로비저닝 (EKS, RDS, S3 …)
  → Ansible: 노드 OS 설정, 에이전트 설치 (필요 시)
  → CD: Kubernetes Deployment 롤링 업데이트
```

## 관련
- [[CICD]] · [[Docker]] · [[Kubernetes]]
