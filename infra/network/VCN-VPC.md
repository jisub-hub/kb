---
tags:
  - network
  - vcn
  - vpc
  - cloud
created: 2026-06-15
---

# 클라우드 가상 네트워크 — VCN / VPC / VNet

> [!summary] 한 줄 요약
> **클라우드 위에 만드는 나만의 격리된 가상 데이터센터 네트워크**. 내부 IP 대역·서브넷·라우팅·게이트웨이를 직접 설계한다. OCI=VCN, AWS=VPC, Azure=VNet.

---

## 1. 왜 필요한가

- 클라우드에서 VM을 만들면 기본적으로 **같은 VCN 안의 VM끼리만 통신** 가능
- 인터넷 노출 범위, 내부 트래픽 경로, 보안 경계를 **직접 제어**해야 운영 환경에 맞는 아키텍처를 구성할 수 있다
- 온프레미스 데이터센터의 네트워크 설계를 클라우드에서 소프트웨어적으로 재현한 것

---

## 2. VCN 핵심 구성 요소

```
VCN (10.0.0.0/16)
├── Public Subnet  (10.0.1.0/24)
│   ├── Route Table: 0.0.0.0/0 → IGW
│   ├── NSG/NACL: 80/443 인바운드 허용
│   └── VM (공인 IP 할당 가능)
│
├── Private Subnet (10.0.2.0/24)
│   ├── Route Table: 0.0.0.0/0 → NAT GW
│   ├── NSG/NACL: 인터넷 인바운드 차단
│   └── VM (사설 IP만)
│
└── DB Subnet      (10.0.3.0/24)
    ├── Route Table: 내부 통신만 (NAT GW 없음 권장)
    ├── NSG/NACL: WAS 서브넷 IP로만 허용
    └── DB VM
```

---

## 3. Public Subnet vs Private Subnet

| | Public Subnet | Private Subnet |
|---|---|---|
| 인터넷 인바운드 | ✅ IGW를 통해 가능 | ❌ 직접 접근 불가 |
| 인터넷 아웃바운드 | ✅ IGW | ✅ NAT GW (단방향) |
| 공인 IP 할당 | 가능 | 불가 (사설 IP만) |
| 배치 대상 | Load Balancer, Bastion Host, 웹서버(간단 구성) | WAS, App Server, DB |
| 보안 수준 | 낮음 → ACG 필수 강화 | 높음 (인터넷 직접 노출 없음) |

---

## 4. Route Table (라우팅 테이블)

서브넷의 트래픽 경로를 결정한다. "목적지 IP → 어디로 보낼지" 규칙 목록.

**Public Subnet 라우팅 테이블 예시:**
| 목적지 | 게이트웨이 | 설명 |
|---|---|---|
| 10.0.0.0/16 | local | VCN 내부 통신 |
| 0.0.0.0/0 | IGW | 그 외 모든 트래픽 → 인터넷 |

**Private Subnet 라우팅 테이블 예시:**
| 목적지 | 게이트웨이 | 설명 |
|---|---|---|
| 10.0.0.0/16 | local | VCN 내부 통신 |
| 0.0.0.0/0 | NAT GW | 아웃바운드 인터넷 (단방향) |

**DB Subnet 라우팅 테이블 (인터넷 아웃바운드도 차단하고 싶을 때):**
| 목적지 | 게이트웨이 | 설명 |
|---|---|---|
| 10.0.0.0/16 | local | VCN 내부 통신만 |

> DB는 패키지 업데이트 시에도 NAT GW를 달지 않고 **Bastion Host를 통한 내부 접근**으로 관리하는 것이 보안상 권장된다.

---

## 5. 클라우드 벤더별 명칭 비교

| 개념 | OCI | AWS | Azure |
|---|---|---|---|
| 가상 네트워크 | VCN | VPC | VNet |
| 인터넷 게이트웨이 | Internet Gateway | Internet Gateway | Internet Gateway |
| NAT 게이트웨이 | NAT Gateway | NAT Gateway | NAT Gateway |
| 클라우드 서비스 GW | Service Gateway | Gateway Endpoint | Service Endpoint |
| 인스턴스 방화벽 | ACG (Security Group) | Security Group | NSG |
| 서브넷 방화벽 | NSG (구 Security List) | Network ACL | — (NSG가 VM 단위) |
| VCN 간 연결 | LPG | VPC Peering | VNet Peering |
| 온프레미스 연결 | DRG | VGW + VPN/DX | VPN GW + ExpressRoute |

---

## 6. 관련
- [[Subnetting]] · [[Gateway]] · [[Security-ACG-NSG]] · [[HA-Architecture]]
