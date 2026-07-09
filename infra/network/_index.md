---
tags:
  - infra
  - network
  - moc
  - index
created: 2026-06-15
---

# Network MOC

> 클라우드 인프라를 이해하는 데 필요한 네트워크 기초부터 보안 설계까지.

## 파일 구성
- [[Network-Basics]] — IP 주소체계, OSI/TCP-IP 계층, 주요 프로토콜
- [[Routing]] — 라우팅 테이블, Hop·TTL, 정적/동적 라우팅, Port Forwarding, SSH 터널링
- [[Subnetting]] — CIDR 표기법, 서브넷 마스크, 역할별 분할 설계
- [[VCN-VPC]] — 클라우드 가상 네트워크 (OCI VCN / AWS VPC / Azure VNet)
- [[Gateway]] — IGW · NAT GW · Service GW · DRG · LPG · IPSec VPN · FastConnect/DirectConnect/ExpressRoute
- [[VPN]] — IPSec Site-to-Site · SSL VPN · WireGuard · 선택 기준
- [[Load-Balancer]] — L4 NLB / L7 ALB, 알고리즘, Health Check, SSL Termination
- [[API-Gateway]] — API GW vs LB 차이, 인증·Rate Limit·라우팅
- [[Security-ACG-NSG]] — ACG/SG(인스턴스) · NSG/NACL(서브넷) · VM OS 방화벽 · Defense in Depth
- [[HA-Architecture]] — 단일화 vs 이중화, Cross-Connected 패턴, Failover
- [[Bandwidth]] — 대역폭·처리량·지연 지표, 클라우드 비용, CDN
- [[BGP-Anycast]] — BGP 라우팅 프로토콜, Anycast 원리, DDoS 방어 활용

## 핵심 흐름
```
인터넷
  → IGW (VCN 진입)
  → NSG/NACL (서브넷 1차 필터)
  → Load Balancer (트래픽 분산)
  → ACG/SG (VM 2차 필터)
  → VM OS 방화벽 (최후 방어선)
  → Application
```

## 관련
- [[../ssh-cicd/_index|SSH & CI/CD]] · [[../terraform/Terraform|Terraform]] · [[../k8s/Kubernetes|Kubernetes]]
