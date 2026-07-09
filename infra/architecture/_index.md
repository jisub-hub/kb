---
tags:
  - infra
  - architecture
  - moc
  - index
created: 2026-06-15
---

# 인프라 아키텍처 MOC

> 실제 서비스 환경에서 쓰이는 주요 인프라 아키텍처 패턴 정리.

## 파일 구성
- [[3-Tier-Architecture]] — 전통적 Web/WAS/DB 3계층 구조, 서브넷 분리, HA 설계
- [[K8s-Architecture]] — 쿠버네티스 기반 컨테이너 운영 환경
- [[MSA-Architecture]] — MSA 환경, 서비스 분리 기준, 통신 패턴, 분산 시스템 이슈
- [[Public-Network-Security]] — 공공망/업무망 환경: 프론트 포함 전 서버 Private, LB 단일 진입점

## 진화 흐름
```
모놀리스 → 3-Tier → MSA → Kubernetes 기반 MSA
```

## 관련
- [[../network/_index|Network]] · [[../k8s/_index|Kubernetes]] · [[../../programming/spring/architecture/_index|Spring Architecture]]
