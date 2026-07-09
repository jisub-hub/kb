---
tags:
  - network
  - gateway
  - igw
  - nat
  - cloud
created: 2026-06-15
---

# 게이트웨이 종류 — IGW · NAT GW · Service GW · DRG · LPG

> [!summary] 한 줄 요약
> 게이트웨이는 VCN/VPC의 **트래픽 출입구**. 목적(인터넷 양방향/단방향/클라우드 서비스/온프레미스)에 따라 다른 게이트웨이를 사용한다.

---

## 1. Internet Gateway (IGW) — VCN ↔ 인터넷 양방향

```
인터넷 ←→ IGW ←→ Public Subnet VM (공인 IP 필요)
```

- **양방향** 통신 (인바운드 + 아웃바운드)
- Public Subnet의 VM이 공인 IP를 가져야 통신 가능
- Route Table에 `0.0.0.0/0 → IGW` 설정
- 배치 대상: 웹서버, Load Balancer, Bastion Host

> [!danger] IGW가 붙은 서브넷은 인터넷에 직접 노출
> ACG/NSG 설정이 느슨하면 전 세계에서 접근 가능 → 최소 권한 원칙 필수

---

## 2. NAT Gateway — Private Subnet → 인터넷 단방향

```
Private Subnet VM → NAT GW → 인터넷 (아웃바운드만)
인터넷 → Private Subnet VM (×, 불가)
```

- **단방향 아웃바운드만** 허용. 인터넷에서 내부로 직접 접근 불가
- VM에 공인 IP 없이 외부 통신 가능 (NAT GW 자체가 공인 IP 보유)
- 주 용도: Private Subnet VM의 **패키지 업데이트, 외부 API 호출**
- Route Table: `0.0.0.0/0 → NAT GW`

**IGW vs NAT GW 비교:**

| | IGW | NAT GW |
|---|---|---|
| 방향 | 양방향 | 아웃바운드 전용 |
| 인바운드 허용 | ✅ (ACG로 제어) | ❌ |
| 공인 IP | VM에 직접 할당 | NAT GW가 보유 |
| 배치 서브넷 | Public | Private |

---

## 3. Service Gateway (SGW) — Private VM → 클라우드 서비스

```
Private Subnet VM → Service GW → OCI Object Storage / AWS S3 / ...
(인터넷 경유 없음)
```

- **인터넷을 거치지 않고** 클라우드 관리 서비스에 접근
- OCI: Object Storage, Autonomous DB, Monitoring 등
- AWS: S3, DynamoDB (Gateway Endpoint)
- Azure: Storage Account, SQL (Service Endpoint)

**장점:**
- 보안: 데이터가 퍼블릭 인터넷 미경유
- 비용: 인터넷 아웃바운드 트래픽 비용 절감
- 속도: 클라우드 백본 사용 → 지연 감소

---

## 4. DRG (Dynamic Routing Gateway) — VCN ↔ 온프레미스

```
VCN ←→ DRG ←→ VPN 터널 or FastConnect ←→ 온프레미스 IDC
```

- **하이브리드 클라우드** 구성 시 사용
- OCI: DRG / AWS: Virtual Private Gateway / Azure: VPN Gateway
- 연결 방식: **IPSec VPN** (인터넷 경유, 암호화) 또는 **전용선** (FastConnect/Direct Connect/ExpressRoute)

| 연결 방식 | 대역폭 | 비용 | 안정성 |
|---|---|---|---|
| IPSec VPN | 낮음 (인터넷 의존) | 저렴 | 인터넷 품질에 의존 |
| 전용선 | 최대 수십 Gbps | 높음 | 안정적, SLA 보장 |

---

## 5. LPG (Local Peering Gateway) — VCN ↔ VCN

```
VCN-A ←→ LPG ←→ LPG ←→ VCN-B (같은 리전)
```

- 같은 리전 내 **VCN 간 직접 연결** (인터넷 미경유)
- OCI: LPG / AWS: VPC Peering / Azure: VNet Peering
- 용도: 서비스별 VCN 분리 후 필요한 VCN끼리만 연결

> [!warning] VPC Peering 제약
> Peering은 **비전이적(non-transitive)**. A↔B, B↔C가 연결되어도 A↔C는 직접 통신 불가. A↔C도 별도 Peering 필요.
> 다수 VCN을 허브처럼 연결할 때는 **Transit Gateway** (AWS) / **DRG 허브** (OCI) 방식을 사용.

---

## 6. 게이트웨이 선택 요약

| 상황 | 사용할 게이트웨이 |
|---|---|
| Public Subnet VM이 인터넷과 통신 | IGW |
| Private Subnet VM이 외부 API/패키지 업데이트 | NAT GW |
| Private Subnet VM이 S3/Object Storage 접근 | Service GW |
| 온프레미스 IDC와 VCN 연결 | DRG (VPN or 전용선) |
| 다른 VCN과 내부망 연결 | LPG |

---

## 7. IPSec VPN vs 전용선 상세 비교

DRG를 통해 온프레미스와 클라우드를 연결할 때 두 가지 방식 중 선택.

### IPSec VPN (인터넷 경유)

```
온프레미스 라우터 ──IPSec 터널(인터넷)──→ DRG (클라우드)
```

```
클라우드에서 설정:
1. DRG 생성 → VCN에 연결
2. IPSec Connection 생성
   - 온프레미스 공인 IP 입력
   - Pre-Shared Key (공유 비밀키) 설정
   - IKE 버전: IKEv2 권장
   - 암호화: AES-256, SHA-256
3. Tunnel 2개 자동 생성 (HA — 하나 장애 시 자동 전환)
4. 온프레미스 방화벽/라우터에 동일 설정
5. 라우팅: 온프레미스 대역 → DRG
```

| 항목 | 내용 |
|---|---|
| 비용 | 저렴 (인터넷 회선 사용) |
| 대역폭 | 인터넷 품질에 의존 (수백 Mbps 수준) |
| 지연 | 인터넷 경로 의존 (변동 있음) |
| 보안 | AES-256 암호화 → 도청 방지 |
| 가용성 | 인터넷 장애 영향 받음 |
| 적합한 상황 | 비용 우선, 개발/테스트 환경, 소규모 데이터 전송 |

---

### 전용선 연결 (물리 직결)

공용 인터넷을 거치지 않고 **통신사 전용 회선**으로 클라우드와 직결.

| 클라우드 | 전용선 서비스 |
|---|---|
| OCI | **FastConnect** |
| AWS | **Direct Connect** |
| Azure | **ExpressRoute** |
| GCP | Cloud Interconnect |

```
온프레미스 IDC ──전용 광회선── 통신사 POP (IX) ──전용선──→ 클라우드 DRG
                              (코로케이션 시설)
```

#### FastConnect (OCI) 설정

```
1. OCI 콘솔에서 FastConnect Virtual Circuit 생성
2. 파트너사(KT, SKT, LG U+ 등) 통해 전용 회선 계약
3. BGP 피어링 설정 (ASN, 피어 IP 교환)
4. VCN Route Table에 온프레미스 대역 → DRG 추가
```

#### 전용선 특성

| 항목 | 내용 |
|---|---|
| 비용 | 높음 (월 수백만 원~) |
| 대역폭 | 100Mbps ~ 100Gbps |
| 지연 | 안정적, 예측 가능 (SLA 보장) |
| 보안 | 공용망 미경유 → 도청 위험 없음 |
| 가용성 | 높음 (SLA 99.9%+, 이중화 가능) |
| 적합한 상황 | 금융/의료 규제 환경, 대용량 데이터 이전, 하이브리드 프로덕션 |

---

### IPSec VPN vs 전용선 선택 기준

| 상황 | 선택 |
|---|---|
| 비용 최우선, 소규모 | IPSec VPN |
| 개발·테스트 환경 | IPSec VPN |
| 금융·의료 규제 준수 (보안 필수) | 전용선 |
| 대용량 데이터 마이그레이션 (TB+) | 전용선 |
| 레이턴시 SLA 있는 프로덕션 | 전용선 |
| 재해복구(DR) 사이트 | IPSec VPN (비용) 또는 전용선 (SLA) |

---

## 8. 관련
- [[VCN-VPC]] · [[Subnetting]] · [[HA-Architecture]] · [[VPN]]
