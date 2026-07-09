---
tags:
  - hardware
  - server
  - blade
  - datacenter
  - dell
  - hpe
  - supermicro
created: 2026-06-16
---

# Blade Server

> [!summary] 한 줄 요약
> 블레이드 서버는 **전원·냉각·네트워킹을 섀시(Chassis)에서 공유**하고, 컴퓨트 블레이드만 꽂아 쓰는 모듈형 서버. 고밀도·중앙 관리·케이블 최소화가 강점. 초기 섀시 비용이 높아 10대 이상 운영 시 TCO 이점이 발생.

---

## 1. 블레이드 서버 구조

```
┌─────────────────────────────────────────────────────┐
│                  Blade Chassis (7U~10U)              │
│                                                      │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐         │
│  │ B1 │ │ B2 │ │ B3 │ │ B4 │ │ B5 │ │ B6 │  ...    │
│  │CPU │ │CPU │ │CPU │ │CPU │ │CPU │ │CPU │  블레이드  │
│  │RAM │ │RAM │ │RAM │ │RAM │ │RAM │ │RAM │  (컴퓨트) │
│  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘         │
│                                                      │
│  [공유 전원 공급 장치]  [공유 팬/냉각 모듈]            │
│  [공유 패브릭 스위치]   [관리 모듈 (CMC)]              │
└─────────────────────────────────────────────────────┘

블레이드 서버 vs 랙 서버:
  랙 서버: 각 서버가 독립적인 전원·팬·NIC 보유
  블레이드: 섀시 내 모든 블레이드가 공유 → 케이블 대폭 감소
```

### 핵심 구성 요소

| 구성 요소 | 역할 | 비고 |
|---|---|---|
| **Chassis (섀시)** | 공유 인프라 제공 (전원, 냉각, 패브릭) | 7~10U 크기 |
| **Blade (컴퓨트 블레이드)** | CPU, RAM, 로컬 스토리지 탑재 | 슬라이드인 방식 |
| **CMC (Chassis Management Controller)** | 섀시·블레이드 통합 원격 관리 | 단일 관리 포인트 |
| **패브릭 모듈 (Fabric Module)** | 블레이드간·외부 네트워크 스위칭 | 10GbE/25GbE/InfiniBand |
| **전원 공급 모듈** | N+N 이중화 PSU | 섀시 공유 |
| **냉각 모듈** | 공유 팬 트레이 | 블레이드 밀도 대비 효율 |

---

## 2. 주요 벤더 및 제품

### 2.1 Dell PowerEdge MX 시리즈

```
섀시:  MX7000 (10U, 최대 8개 전폭 블레이드 / 16개 반폭)
관리:  OpenManage Enterprise + iDRAC (통합 관리)
패브릭: SmartFabric OS — 자동 네트워크 설정 (Ethernet/FC)
특징:  "Kinetic Infrastructure" — 블레이드 간 리소스 재할당 지원
```

**MX7000 블레이드 라인업:**

| 블레이드 | 소켓 | CPU | 최대 RAM | 스토리지 | 특징 |
|---|---|---|---|---|---|
| **MX740c** | 2S | Xeon Scalable 4세대 | 6TB DDR5 | NVMe/SAS | 범용 고성능 |
| **MX760c** | 2S | Xeon Scalable 4세대 | 8TB DDR5 | NVMe | 메모리 집약 |
| **MX750c** | 2S | Xeon Scalable 3세대 | 3TB DDR4 | SAS/NVMe | 전 세대 |
| **MXG610s** | — | — | — | 25GbE 스위치 | 패브릭 모듈 |

**특징:**
- SmartFabric: NIC→Switch 자동 VLAN 설정, STP 불필요
- 블레이드 Hot-Plug: 전원 오프 없이 교체
- 단일 CMC로 최대 4섀시(32 블레이드) 관리

---

### 2.2 HPE Synergy (컴포저블 인프라)

```
섀시:  Synergy 12000 Frame (10U, 최대 12 컴퓨트 모듈)
관리:  HPE OneView — 정책 기반 인프라 자동화
패브릭: Virtual Connect SE / SAS Module
핵심 철학: "Composable Infrastructure"
  → CPU+메모리+스토리지+네트워크를 소프트웨어로 재구성
```

**Synergy 컴퓨트 모듈:**

| 모듈 | CPU | 최대 RAM | GPU 지원 | 특징 |
|---|---|---|---|---|
| **SY480 Gen11** | Xeon 4세대 2S | 8TB DDR5 | 최대 2× GPU | 범용, 가장 많이 사용 |
| **SY660 Gen11** | Xeon 4세대 2S | 12TB DDR5 | 없음 | 메모리 집약 (SAP HANA) |
| **SY480 Gen10 Plus** | Xeon 3세대 2S | 6TB DDR4 | 최대 2× GPU | 전 세대 |

**특징:**
- **Composable**: CPU 모듈 + 스토리지 모듈을 논리적으로 조합 (물리 이동 없음)
- HPE OneView: API 기반 IaC 가능 (Ansible/Terraform 연동)
- 직접 수냉(DLC) 옵션으로 고밀도 열 관리
- Fluid Resource Pools: GPU·FPGA를 여러 블레이드가 공유 가능 (PCIe Fabric)

---

### 2.3 Supermicro SuperBlade

```
섀시:  SBE-710E (7U, 최대 20개 블레이드)
       SBE-820E (8U, 최대 20개 블레이드)
관리:  IPMI 2.0 + Supermicro BMC
패브릭: 1/10/25/100GbE 패스스루/스위치 선택
특징:  가성비, 밀도 최고 (20 블레이드/7U)
```

**SuperBlade 블레이드 라인업:**

| 블레이드 | CPU | RAM | 특징 |
|---|---|---|---|
| **SBI-6119R** | Xeon 3세대 2S | 3TB DDR4 | 범용 |
| **SBI-6229R** | Xeon 3세대 2S | 6TB DDR4 | 고메모리 |
| **SBI-420P** | Xeon 4세대 2S | 4TB DDR5 | 최신 세대 |
| **SBI-420P-1C2N** | Xeon 4세대 2S | 4TB DDR5 | 25GbE 내장 |

**특징:**
- 20 블레이드/7U = 랙 서버 대비 최고 밀도
- 가격 경쟁력 (Dell/HPE 대비 30~40% 저렴)
- 관리 툴 성숙도는 낮음 (IPMI 수준)
- OCP(Open Compute) 준수

---

### 2.4 Lenovo ThinkSystem SN 시리즈

```
섀시:  SD665 V3 (2U, 4-in-2U 수냉 특화)
       SN550 (10U, 최대 8 블레이드)
특징:  Neptune 수냉 기술로 GPU 블레이드 지원
       Top500 슈퍼컴퓨터 구축 레퍼런스
```

| 블레이드 | CPU | 냉각 | 특징 |
|---|---|---|---|
| **SN550 V2** | Xeon 3세대 2S | 공랭 | 범용 블레이드 |
| **SD665 V3** | AMD EPYC 9004 2S | **수냉** | 4-in-2U 고밀도 HPC |
| **SD650 V3** | Xeon 4세대 2S | **수냉** | Neptune DLC, HPC |

---

## 3. 블레이드 vs 랙 서버 비교

| 항목 | 블레이드 서버 | 랙 서버 |
|---|---|---|
| **밀도** | 매우 높음 (20서버/7U) | 보통 (42서버/42U = 1U 기준) |
| **케이블 수** | 매우 적음 (섀시 하나로 집결) | 매우 많음 (서버마다 별도) |
| **초기 비용** | 높음 (섀시 $10K~$30K+) | 낮음 (서버 단위 구매) |
| **확장성** | 섀시 내 블레이드 추가 (빠름) | 서버 단위 추가 (유연) |
| **GPU 지원** | 제한적 (슬림 폼팩터) | 우수 (4U에 8× GPU) |
| **관리** | 중앙화 (CMC 단일 포인트) | 분산 (서버마다 BMC) |
| **전력 효율** | 높음 (공유 PSU 이중화) | 보통 (서버마다 PSU) |
| **벤더 종속** | 높음 (섀시 벤더 종속) | 낮음 (혼용 가능) |
| **냉각** | 섀시 공유, 제약 있음 | 서버별 독립, 유연 |
| **TCO 손익분기** | 10대 이상부터 유리 | 소규모에 유리 |

---

## 4. 블레이드 서버 적합 워크로드

```
[적합]
  ✅ 고밀도 가상화 (VMware vSphere, KVM)
  ✅ VDI (Virtual Desktop Infrastructure) — 많은 세션, 공간 제약
  ✅ 웹/앱 서버 클러스터 (동일 사양 대량)
  ✅ HPC 노드 (MPI 클러스터, 고속 패브릭 활용)
  ✅ CI/CD 빌드 팜 (빠른 블레이드 교체)
  ✅ 금융 거래 시스템 (저지연 패브릭)

[부적합]
  ❌ GPU 집약 AI 학습 (GPU 슬롯 수 제한)
  ❌ 스토리지 집약 워크로드 (드라이브 슬롯 제한)
  ❌ 소규모 (10대 미만) — 섀시 초기비용 회수 어려움
  ❌ 멀티벤더 혼용 환경 — 섀시 벤더 종속
```

---

## 5. 블레이드 서버 도입 체크리스트

```
□ 필요 서버 수 > 10대 (TCO 손익분기)
□ 랙 공간 제약 있음 (코로케이션 비용 높음)
□ 동일/유사 사양 워크로드 (이기종 최소화)
□ 중앙화된 관리 필요 (CMC/OneView)
□ GPU 워크로드 비중 낮음 (또는 HPE Synergy PCIe Fabric 사용)
□ 장기 운영 계획 (5년+ 섀시 활용)
□ 네트워크 엔지니어 확보 (패브릭 설계 필요)
```

---

## 6. 벤더 선택 가이드

| 조건 | 권장 벤더·제품 | 이유 |
|---|---|---|
| 엔터프라이즈 통합 관리 | **Dell PowerEdge MX7000** | SmartFabric, OpenManage, iDRAC |
| 컴포저블/자동화 인프라 | **HPE Synergy 12000** | OneView API, Composable |
| 가성비 고밀도 | **Supermicro SuperBlade** | 20-in-7U, IPMI |
| HPC/수냉 고밀도 | **Lenovo SD665 V3** | Neptune 수냉, EPYC |
| 공공기관·금융 (국내) | **Dell MX / HPE Synergy** | 국내 지원·인증 체계 우수 |

---

## 7. 관련
- [[../rack-mount-server/Rack-Mount-Server]] · [[../gpu-workstation/_index]]
- [[../../infra/architecture/MSA-Architecture]]
