---
tags:
  - hardware
  - server
  - rack
  - datacenter
  - dell
  - hpe
  - supermicro
created: 2026-06-16
---

# Rack Mount Server

> [!summary] 한 줄 요약
> 랙 마운트 서버는 표준 19인치 랙에 장착하는 서버. **1U(초고밀도)→2U(균형)→4U(고성능)**으로 높이가 늘수록 확장성이 커진다. 폼팩터 선택은 공간 비용·냉각·GPU 슬롯 수로 결정.

---

## 1. 랙 유닛(U) 기초

```
1U = 1.75인치 (44.45mm) 높이
표준 랙 = 42U 총 높이 = 73.5인치 (186.6cm)

폼팩터  높이(U)  높이(인치)  활용
────────────────────────────────────────────────
1U      1        1.75"     고밀도, 저장장치 제한
2U      2        3.50"     범용, 엔터프라이즈 표준
3U      3        5.25"     틈새, 스토리지 특화
4U      4        7.00"     GPU 서버, 대규모 스토리지
8U+     8+       14"+      GPU 클러스터, 슈퍼컴퓨터
```

---

## 2. 폼팩터별 특징

### 2.1 1U 서버

```
[특징]
높이: 1.75인치 (1U)
CPU:  일반적으로 1~2소켓 지원
RAM:  16~32 DIMM 슬롯 (최대 2~4TB)
드라이브: 4~10개 (2.5" SFF 위주)
PCIe:  1~2 슬롯 (낮은 프로파일 카드만)
GPU:  제한적 (단일 LP GPU만, AI GPU 불가)

[장점]
- 공간 효율 최고 (42U 랙에 42대)
- 코로케이션 공간 비용 절감
- 전력 밀도 균일

[단점]
- 냉각 팬 소음 심함 (팬이 작아 고RPM)
- GPU 확장 거의 불가
- 스토리지 용량 제한

[주요 용도]
웹서버, DNS, 로드밸런서, 마이크로서비스, CDN 엣지
```

**주요 제품:**
| 벤더 | 모델 | CPU | 최대 RAM | 특징 |
|---|---|---|---|---|
| **Dell** | PowerEdge R660 | Xeon 4세대 2S | 4TB DDR5 | 엔터프라이즈 표준 |
| **HPE** | ProLiant DL360 Gen11 | Xeon 4세대 2S | 4TB DDR5 | ISS 인증, iLO 관리 |
| **Supermicro** | SYS-110P-WTR | Xeon 3세대 1S | 2TB | 컴팩트, 가성비 |
| **Lenovo** | ThinkSystem SR630 V3 | Xeon 4세대 2S | 4TB DDR5 | SR IOV, 고신뢰 |

---

### 2.2 2U 서버

```
[특징]
높이: 3.5인치 (2U)
CPU:  1~2소켓 (주류 엔터프라이즈)
RAM:  24~32 DIMM 슬롯 (최대 4~8TB)
드라이브: 8~24개 (2.5" SFF 또는 3.5" LFF)
PCIe:  4~8 슬롯 (LP + HL 카드)
GPU:  1~2개 (풀사이즈 단일너비)

[장점]
- 확장성과 밀도의 균형
- 3.5" LFF 드라이브 수용 가능 → 고용량 스토리지
- 다양한 PCIe 카드 수용
- 냉각 여유

[단점]
- 1U 대비 공간 2배
- 고성능 GPU 서버로는 부족 (4U 대비)

[주요 용도]
DB 서버, 가상화(VMware/KVM), ERP, 파일서버, 중급 AI 추론
```

**주요 제품:**
| 벤더 | 모델 | CPU | 최대 RAM | 드라이브 | 특징 |
|---|---|---|---|---|---|
| **Dell** | PowerEdge R760 | Xeon 4세대 2S | 8TB DDR5 | 최대 24× 2.5" | 엔터프라이즈 No.1 |
| **HPE** | ProLiant DL380 Gen11 | Xeon 4세대 2S | 8TB DDR5 | 최대 24× 2.5" | 업계 베스트셀러 |
| **Supermicro** | SYS-220P-C9R | Xeon 3세대 2S | 6TB | 최대 24× 2.5" | 고가성비 |
| **Lenovo** | ThinkSystem SR650 V3 | Xeon 4세대 2S | 8TB DDR5 | 최대 32× 2.5" | Anycloud 지원 |

---

### 2.3 4U 서버

```
[특징]
높이: 7인치 (4U)
CPU:  2~8소켓 (8소켓은 특수)
RAM:  48~96 DIMM 슬롯 (최대 24TB+)
드라이브: 24~60개 (JBOD 구성 가능)
PCIe:  8~16+ 슬롯 (더블너비 GPU 수용)
GPU:  4~8개 (H100/A100 급 풀사이즈 지원)

[장점]
- 고성능 GPU 다수 수용 핵심 폼팩터
- 대용량 스토리지 (60× 3.5" → 1PB급)
- 고전력 GPU의 냉각 공간 충분
- 액체냉각 시스템 탑재 가능

[단점]
- 랙 공간 4U 소모 → 밀도 낮음
- 고전력 (GPU 탑재 시 5,000~10,000W/서버)
- 무거움 (50~80kg)

[주요 용도]
AI 학습·추론 서버, HPC, 빅데이터, OLAP, 가상화 대규모
```

**주요 제품:**
| 벤더 | 모델 | CPU | GPU | 특징 |
|---|---|---|---|---|
| **Dell** | PowerEdge R760xa | Xeon 4세대 2S | 최대 4× H100 | AI 추론 최적화 |
| **Dell** | PowerEdge XE8640 | Xeon 4세대 2S | 최대 4× H100 SXM | 고성능 AI 학습 |
| **HPE** | ProLiant DL380 Gen11 (4U) | Xeon 4세대 2S | 최대 4× A100 | 범용 고성능 |
| **HPE** | ProLiant DL380a Gen11 | Xeon 4세대 2S | 최대 4× H100 | AI 특화 |
| **Supermicro** | SYS-421GE-TNRT | Xeon 4세대 2S | 최대 8× A100 | 최고 밀도 GPU |
| **Lenovo** | ThinkSystem SR670 V3 | Xeon 4세대 2S | 최대 8× H100 | HPC 특화 |

---

## 3. 벤더 비교

### 3.1 Dell Technologies (PowerEdge)

```
강점:
  · 글로벌 No.1 서버 점유율 (약 18~20%)
  · OpenManage / iDRAC 관리 툴 성숙도 최고
  · 광범위한 부품 수급·유지보수 네트워크
  · PowerEdge XE 시리즈: AI/HPC GPU 서버 특화
  · APEX (as-a-Service) 클라우드 소비 모델 지원

약점:
  · 가격 프리미엄 (HPE/Supermicro 대비 10~20% 높음)
  · 독점 부품 종속성

주요 라인업:
  R-series  — 범용 랙 (R660=1U, R760=2U, R960=4U 8S)
  XE-series — AI/HPC 특화 (XE8640 4U GPU, XE9712 GB200)
  MX-series — 블레이드 플랫폼
```

### 3.2 HPE (ProLiant / Compute)

```
강점:
  · iLO(Integrated Lights-Out) 원격 관리 성숙
  · 액체 냉각(Direct Liquid Cooling) 기술 선도
  · HPE Synergy: 컴포저블 인프라
  · GreenLake: 서비스형 인프라 (종량제)
  · 슈퍼컴퓨터(Frontier 등) 구축 경험

약점:
  · 복잡한 SKU 체계 (제품 라인 너무 많음)
  · 고가 (특히 Synergy)
  · 설정 복잡도 높음

주요 라인업:
  DL-series  — 랙 서버 (DL360=1U, DL380=2U, DL580=4U)
  XD-series  — AI 가속 서버 (XD685 AMD MI325X)
  Synergy    — 컴포저블 블레이드
```

### 3.3 Supermicro

```
강점:
  · 가격 경쟁력 (Dell/HPE 대비 20~30% 저렴)
  · GPU 서버 다양성 (HGX A100/H100/B200 커스텀 폼팩터)
  · OCP(Open Compute Project) 준수 → 오픈 생태계
  · 빠른 신제품 출시 (B200 최초 양산 2025.2)
  · 하드웨어 커스터마이징 유연

약점:
  · 엔터프라이즈 지원 체계 상대적으로 약함
  · 관리 소프트웨어 성숙도 낮음 (IPMI 기반)
  · 브랜드 인지도

주요 라인업:
  A+/B+/X13 시리즈 — 범용
  SYS-4U  — GPU 고밀도 (8× GPU)
  HGX 플랫폼  — NVIDIA HGX B200/H100 공식 파트너
```

### 3.4 Lenovo (ThinkSystem)

```
강점:
  · 신뢰성·안정성 평판 (IBM 서버 사업 계승)
  · Neptune 수냉 기술 (직접 수냉, 45°C 냉각수 지원)
  · HPC/슈퍼컴퓨터 경험 (Top500 다수)
  · SD650 V3: 고밀도 수냉 4-in-2U

약점:
  · 아시아 중심 (북미·유럽 점유율 낮음)
  · 에코시스템 델/HPE 대비 좁음

주요 라인업:
  SR-series  — 범용 랙 (SR630=1U, SR650=2U, SR670=4U)
  SD-series  — 고밀도 수냉 (SD650 V3)
  SE-series  — 엣지 서버
```

---

## 4. 폼팩터 선택 기준

```
[결정 트리]

GPU가 주요 워크로드인가?
  YES → 4U (GPU 4~8개) 또는 8U+ (HGX 시스템)
  NO  →
    스토리지 중심인가?
      YES → 2U (24× LFF) 또는 스토리지 전용 서버
      NO  →
        밀도 최우선 (코로케이션 비용)?
          YES → 1U
          NO  → 2U (엔터프라이즈 범용 최적)
```

| 워크로드 | 권장 폼팩터 | 권장 벤더 |
|---|---|---|
| 웹/API 서버, 마이크로서비스 | **1U** | Dell R660, HPE DL360 |
| DB, VM, ERP | **2U** | Dell R760, HPE DL380 |
| AI 추론 (소규모) | **2U** (GPU 1~2개) | Dell R760xa, HPE DL380a |
| AI 학습·대규모 추론 | **4U** (GPU 4~8개) | Supermicro SYS-421GE, Dell XE8640 |
| 대용량 스토리지 | **4U** (JBOD) | Supermicro 60-bay, HPE MSA |
| HPC 클러스터 | **4U/수냉** | Lenovo SR670 V3, HPE XD |
| GB200 NVL72 | **랙 전체** | Dell XE9712, Supermicro HGX |

---

## 5. 전력·냉각 고려사항

```
폼팩터별 일반 전력 범위 (GPU 없이):
  1U  :   300~  600W (Dual PSU)
  2U  :   500~1,200W (Dual PSU)
  4U  : 1,000~2,000W (Redundant PSU)

GPU 탑재 시 추가 전력:
  H100 × 4 (SXM5 700W × 4) → +2,800W → 4U 서버 총 4,000W+
  B200 × 4 (1,000W × 4)    → +4,000W → 4U 서버 총 5,500W+

냉각 방식:
  공랭 (Air Cooling)  — 1U/2U 표준, 팬 속도·소음 타협
  직접 수냉 (DLC)     — 4U GPU 서버 필수, GPU에 냉각판 부착
  침지 냉각 (Immersion) — 최고 효율, 데이터센터 전체 개조 필요
```

---

## 6. IPMI / BMC — 원격 하드웨어 관리

**IPMI (Intelligent Platform Management Interface)**: 서버 OS와 독립된 하드웨어 레벨 원격 관리 인터페이스. OS 부팅 전·장애 상태에서도 접근 가능.

```
[구조]
  서버 메인보드
    └── BMC (Baseboard Management Controller) 칩
          ├── 전용 NIC (IPMI LAN 포트) ← 관리 네트워크 분리 권장
          ├── 전용 CPU (ARM 기반, OS와 독립)
          └── 전용 전원 (서버 전원 OFF 상태에서도 동작)

관리 네트워크 분리:
  [프로덕션 네트워크]  ←→  서버 메인 NIC
  [관리 네트워크 (OOB)] ←→  BMC IPMI 포트  ← 인터넷 노출 절대 금지!
```

### 벤더별 IPMI 구현 명칭

| 벤더 | BMC 브랜드 | 특징 |
|---|---|---|
| **Dell** | iDRAC (Integrated Dell Remote Access Controller) | OpenManage 통합, GUI 우수 |
| **HPE** | iLO (Integrated Lights-Out) | HPE OneView 통합, 보안 강함 |
| **Supermicro** | IPMI / IPMICFG | 표준 IPMI 2.0, 저비용 |
| **Lenovo** | XCC (Extreme Cloud Controller) | Fan/전력 정밀 제어 |
| **ASRock/ASUS** | IPMI | 마더보드 내장 |

### 주요 기능

```
[전원 제어]
  원격 전원 ON/OFF/재부팅/강제 전원 차단
  PXE 부팅 강제 (네트워크 부팅으로 OS 재설치)
  BIOS/UEFI 설정 원격 변경

[콘솔 접근 (KVM-over-IP)]
  → 아래 KVM 섹션 참조

[하드웨어 모니터링]
  온도 센서 (CPU, DIMM, PCH, 드라이브)
  팬 RPM 모니터링 및 제어
  전력 소비 실시간 확인
  SEL (System Event Log) — 하드웨어 이벤트 로그

[원격 미디어]
  ISO 파일을 가상 CD/USB로 마운트 → OS 설치
  Virtual Media: iDRAC/iLO에서 지원

[경고 알림]
  SNMP Trap → NMS로 하드웨어 이상 알림
  이메일 알림 (온도 임계치 초과, 팬 장애 등)
```

### IPMI 명령줄 (ipmitool)

```bash
# 전원 제어
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password power status
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password power on
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password power reset
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password power off  # 강제 차단

# 센서 모니터링
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password sdr list
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password sensor list

# 이벤트 로그
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password sel list
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password sel clear

# 부팅 순서 변경 (PXE)
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password chassis bootdev pxe
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password chassis bootdev disk

# SOL (Serial Over LAN) — 텍스트 콘솔
ipmitool -I lanplus -H 192.168.100.10 -U admin -P password sol activate
```

### IPMI 보안 주의사항

```
⚠ IPMI는 설계상 보안 취약점이 많음 (Cipher Suite 0, Rakp 해시 노출 등)

필수 보안 조치:
  1. IPMI 포트를 인터넷에 절대 노출 금지
  2. 전용 관리 VLAN에 격리 (OOB Network)
  3. 기본 비밀번호 즉시 변경 (admin/admin 공장 초기값)
  4. IPMI 2.0 + AES 암호화 사용 (Cipher Suite 17 권장)
  5. SSH/HTTPS만 허용, Telnet/HTTP 비활성화
  6. 펌웨어 최신 버전 유지 (BMC 취약점 패치)
  7. VPN 또는 Bastion Host 경유 접근
```

---

## 7. KVM — 서버 원격 콘솔

**KVM (Keyboard, Video, Mouse)**: 서버의 키보드·모니터·마우스를 원격에서 제어하는 기술. OS 장애·부팅 과정·BIOS 설정 등 OS 없이도 작동.

### 종류

```
[하드웨어 KVM 스위치]  — 물리 장비, 여러 서버 전환
  단일 KVM 스위치에 여러 서버 연결
  관리자가 버튼/핫키로 서버 전환
  브랜드: Raritan, Avocent, Vertiv (Cybex)

[KVM-over-IP]  — 네트워크로 원격 접근
  KVM 스위치에 IP 모듈 추가 → 웹 브라우저/클라이언트로 접근
  또는 IPMI/BMC 내장 (iDRAC, iLO, XCC)

[소프트웨어 KVM]  — OS 기반
  NoMachine, VNC, RDP → OS가 살아있어야 동작
  → 하드웨어 장애 시 무용지물
```

### IPMI 내장 KVM-over-IP (iDRAC/iLO)

```
[iDRAC Virtual Console (Dell)]
  웹 브라우저 → JNLP/HTML5 → 서버 화면 직접 제어
  기능:
    · BIOS 설정 화면 접근
    · POST 화면 확인 (부팅 오류 디버깅)
    · OS 콘솔 (블루스크린, 커널 패닉 확인)
    · Virtual Media (ISO 마운트)
    · 원격 전원 제어

  접근:
    https://192.168.100.10  → iDRAC 웹 UI
    iDRAC Express: 기본 KVM
    iDRAC Enterprise: 고급 KVM + 가상 미디어

[iLO (HPE)]
  HTML5 기반 KVM (Java 불필요, 최신 iLO 5+)
  iLO Standard: 기본 모니터링
  iLO Advanced: KVM + 가상 미디어 + 전력 제어

[XCC (Lenovo)]
  HTML5 KVM 지원
  유연한 팬 속도·전력 캡 설정
```

### 하드웨어 KVM 스위치 — 물리 랙 관리

```
[구성 예시]
  [KVM 스위치 (Raritan Dominion KX III)]
    ├── 서버 1 (Cat6 케이블 또는 USB/VGA 어댑터)
    ├── 서버 2
    ├── 서버 3
    ├── ...
    └── IP 모듈 → 네트워크 → 원격 접근

장점:
  · OS 독립 (하드웨어 레벨)
  · 여러 서버 한 화면에서 전환
  · 물리 콘솔과 동일한 경험

주요 제품:
  Raritan Dominion KX III (엔터프라이즈)
  Vertiv Avocent ACS (시리얼 + KVM)
  ATEN CS 시리즈 (중소규모)
  TESmart (가성비)
```

### 관리 네트워크 설계

```
[권장 OOB (Out-of-Band) 네트워크 토폴로지]

  [인터넷]
       │
  [프로덕션 스위치]  ←──── 서버 메인 NIC (데이터)
       │
  [관리 스위치 (VLAN 분리)]
       ├── IPMI BMC 포트 (서버 1)
       ├── IPMI BMC 포트 (서버 2)
       ├── KVM 스위치 IP 모듈
       └── [Bastion Host / Jump Server]
                 │
            관리자 PC (VPN 필요)

핵심 원칙:
  · 관리 네트워크는 인터넷에서 직접 접근 불가
  · VPN 또는 Bastion Host 경유만 허용
  · IPMI/KVM용 별도 VLAN (VLAN 100 등)
  · 관리 네트워크 스위치 ACL로 접근 제어
```

---

## 8. 관련
- [[../gpu-workstation/_index]] · [[../blade-server/Blade-Server]]
- [[../../infra/architecture/Public-Network-Security]] · [[../../infra/network/VCN-VPC]]
