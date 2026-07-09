---
tags:
  - infra
  - security
  - cloud
  - waf
  - ips
  - av
  - ddos
  - siem
created: 2026-06-16
---

# 클라우드 보안 상품 — 역할과 계층

> [!summary] 한 줄 요약
> 클라우드 보안 상품은 **공격 유형·OSI 계층·탐지 vs 차단** 기준으로 분류된다. WAF(L7 웹), IPS(L3~L4 네트워크), AV/EDR(엔드포인트), DDoS Protection(볼륨), SIEM(통합 가시성) 순으로 쌓는 계층형 방어가 기본.

---

## 1. 전체 구조 — 계층별 보안 상품 배치

```
[인터넷 트래픽]
       │
┌──────▼──────────────────────────────────────────────┐
│  DDoS Protection (L3/L4/L7)                         │
│  · 대용량 볼륨 공격 흡수·스크러빙                       │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│  IPS / IDS (L3~L4 네트워크 계층)                      │
│  · 알려진 공격 시그니처 탐지·차단                        │
│  · 비정상 프로토콜·포트 스캔 탐지                        │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│  WAF — Web Application Firewall (L7 HTTP/HTTPS)     │
│  · SQLi, XSS, RCE, SSRF, OWASP Top 10 차단          │
│  · Bot 탐지·관리                                      │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│  Load Balancer / CDN                                 │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│  서버 / VM / 컨테이너                                   │
│  · AV / EDR — 파일·프로세스·행위 기반 탐지              │
│  · HIDS — 호스트 침입 탐지 (파일 무결성, 로그 분석)        │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│  SIEM — 모든 계층 로그 수집·상관 분석·알림               │
└─────────────────────────────────────────────────────┘
```

---

## 2. WAF (Web Application Firewall)

### 역할

```
HTTP/HTTPS 트래픽을 검사하는 L7 방화벽.
일반 방화벽(IP/포트 차단)으로는 막을 수 없는 웹 공격을 차단.

차단 가능:
  ✅ SQL Injection
  ✅ XSS (Cross-Site Scripting)
  ✅ CSRF
  ✅ RCE (명령 삽입, 파일 포함)
  ✅ Path Traversal
  ✅ SSRF
  ✅ Bot·크롤러 관리
  ✅ API Rate Limiting
  ✅ OWASP Top 10 전반

막기 어려운 것:
  ❌ 논리적 취약점 (IDOR, 잘못된 권한)
  ❌ 제로데이 (시그니처 없음)
  ❌ 암호화된 내부 로직 오류
```

### 동작 방식

```
[모드]
  탐지 모드(Detection)   — 로그만 기록, 차단 안 함 (초기 튜닝)
  차단 모드(Prevention)  — 룰 매칭 시 즉시 차단 (운영)

[룰 엔진]
  OWASP CRS (Core Rule Set) — 공개 표준 룰셋
  Managed Rules             — 벤더가 관리 (최신 CVE 반영)
  Custom Rules              — 서비스 특화 룰

[탐지 방식]
  시그니처   — 알려진 패턴 (SQL 예약어, <script> 등)
  정규식     — 복잡한 패턴 매칭
  점수 기반  — 여러 룰 점수 합산 → 임계치 초과 시 차단 (오탐 감소)
  머신러닝   — 정상 트래픽 베이스라인 학습 후 이탈 탐지
```

### 클라우드 벤더별 WAF 상품

| 벤더 | 상품명 | 특징 |
|---|---|---|
| **AWS** | AWS WAF | CloudFront/ALB/API GW 앞단 배치, 관리형 룰 마켓플레이스 |
| **Azure** | Azure WAF (Front Door / App Gateway) | DRS 2.0 룰셋, 봇 보호 통합 |
| **GCP** | Cloud Armor | DDoS + WAF 통합, Adaptive Protection(ML) |
| **Cloudflare** | WAF | 글로벌 Anycast, Zero-Day 대응 빠름, Workers 통합 |
| **Naver Cloud** | Web Firewall | OWASP 기반, 국내 공공 클라우드 CSAP 지원 |
| **KT Cloud** | WAF | 국내 CC인증 |
| **Akamai** | App & API Protector | 대규모 트래픽·API 보호 특화 |

```
WAF 배치 위치:
  CDN (Cloudflare, CloudFront) → 엣지에서 차단 (오리진 도달 전)
  LB 앞단 (ALB + AWS WAF)     → 리전 단위 차단
  Reverse Proxy (Nginx + ModSecurity) → 온프레미스·자체 구축
```

---

## 3. IPS / IDS

### IDS vs IPS 차이

```
IDS (Intrusion Detection System)  — 탐지만, 알림 발생
IPS (Intrusion Prevention System) — 탐지 + 자동 차단 (인라인 배치)

배치:
  IDS: 네트워크 미러 포트(SPAN) 또는 탭 → 트래픽 복사본 분석
       → 실시간 차단 불가, 사후 분석
  IPS: 트래픽 경로 인라인 배치 → 실시간 차단 가능
       → 오탐(False Positive) 시 정상 트래픽도 차단 위험
```

### 탐지 방식

```
[시그니처 기반 (Signature-based)]
  알려진 공격 패턴 DB와 비교
  장점: 오탐 낮음, 빠름
  단점: 제로데이 탐지 불가, 시그니처 업데이트 필수
  예: Snort 룰, Suricata 룰

[이상 탐지 (Anomaly-based)]
  정상 트래픽 베이스라인 학습
  → 기준 벗어나면 알림 (포트 스캔, 갑작스런 트래픽 급증 등)
  장점: 제로데이 탐지 가능
  단점: 오탐 많음, 튜닝 필요

[정책 기반 (Policy-based)]
  보안 정책 위반 탐지 (예: 비인가 프로토콜 사용)
```

### 탐지 항목

```
네트워크 계층:
  · 포트 스캔 (nmap, masscan)
  · OS 핑거프린팅
  · SYN Flood, UDP Flood 등 DoS 시도
  · 알려진 취약점 익스플로잇 패킷
  · Malware C2 서버 통신 (알려진 IP/도메인)
  · 비인가 프로토콜 (예: 내부망에서 Tor, VPN 우회)
  · 횡적 이동 (Lateral Movement) 탐지

애플리케이션 계층 (NGFW/NGIPS):
  · 애플리케이션 식별 (Netflix vs YouTube vs 비인가 P2P)
  · SSL/TLS 복호화 후 검사
  · 파일 타입 기반 차단 (실행파일 다운로드 차단)
```

### 클라우드 벤더별 IPS/IDS 상품

| 벤더 | 상품명 | 특징 |
|---|---|---|
| **AWS** | GuardDuty | VPC Flow Logs + DNS Logs + CloudTrail 분석, ML 기반 이상탐지, 관리형 |
| **AWS** | Network Firewall | VPC 인라인 IPS, Suricata 호환 룰 |
| **Azure** | Defender for Cloud (formerly Azure Security Center) | IDS + 취약점 평가 통합 |
| **Azure** | Azure Firewall Premium | IDPS 기능 내장, TLS 검사 |
| **GCP** | Cloud IDS | Google이 관리하는 Palo Alto Networks 기반 IDS |
| **Naver Cloud** | IPS | 시그니처+이상탐지, 국내 ISMS-P 대응 |
| **Palo Alto** | VM-Series | NGFW/IPS, 클라우드 마켓플레이스 제공 |
| **Fortinet** | FortiGate VM | NGFW, AWS/Azure/GCP 마켓플레이스 |

---

## 4. AV / EDR (Endpoint Security)

### AV (Anti-Virus / Anti-Malware)

```
[전통 AV — 시그니처 기반]
  악성 파일의 해시값·패턴을 DB와 비교
  장점: 알려진 악성코드 빠른 탐지, 가벼움
  단점: 제로데이, 파일리스 악성코드, 변종 탐지 어려움
  예: ClamAV (오픈소스), 서버 파일 업로드 검사에 사용

[행위 기반 탐지]
  프로세스의 실제 행위 분석
  예: 정상 Word.exe가 → cmd.exe를 실행 → netcat 설치 → 의심
  → EDR로 발전
```

### EDR (Endpoint Detection and Response)

```
[개념]
  엔드포인트(서버·PC)에서 모든 프로세스·파일·네트워크·레지스트리
  이벤트를 수집 → 중앙에서 분석 → 위협 탐지 + 대응

[탐지]
  · 파일리스 공격 (PowerShell, WMI 악용)
  · 랜섬웨어 행위 (대량 파일 암호화)
  · C2 통신 패턴
  · 자격증명 덤프 (lsass.exe 접근)
  · 수평 이동 (PsExec, WMI 원격 실행)
  · 권한 상승 시도

[대응]
  프로세스 격리·종료
  파일 격리 (quarantine)
  네트워크 차단 (호스트 격리)
  포렌식 증거 수집 (IOC)
```

| 벤더 | 상품명 | 특징 |
|---|---|---|
| **AWS** | GuardDuty + Malware Protection | S3·EBS 스캔, EC2 런타임 위협 탐지 |
| **Azure** | Microsoft Defender for Endpoint | 서버 에이전트, 위협 헌팅, XDR |
| **GCP** | Chronicle (Google Security Operations) | EDR + SIEM 통합 |
| **CrowdStrike** | Falcon | 클라우드 네이티브 EDR, 엔드포인트·서버·컨테이너 |
| **SentinelOne** | Singularity | AI 기반, 자율 대응 |
| **Naver Cloud** | Anti-Virus | ClamAV 기반, 서버 파일 스캔 |

---

## 5. DDoS Protection

```
[계층별 보호]
  L3/L4 (볼륨형): UDP Flood, SYN Flood, ICMP Flood
    → 업스트림 스크러빙, Anycast 분산
  L7 (애플리케이션형): HTTP Flood, Slowloris
    → WAF + Rate Limiting + Bot Challenge

[핵심 개념]
  스크러빙 센터 — 공격 트래픽을 우회시켜 정화 후 전달
  Anycast    — 공격 트래픽을 여러 PoP에 분산 (한 곳 포화 방지)
  BGP Blackholing — 공격 IP를 null route로 폐기
```

| 벤더 | 상품명 | 보호 용량 |
|---|---|---|
| **AWS** | Shield Standard (무료) | L3/L4 기본 보호, 자동 |
| **AWS** | Shield Advanced ($3K/월) | L7 포함, 24/7 DRT 팀 |
| **Azure** | DDoS Network Protection | 자동 튜닝, 100Gbps+ |
| **GCP** | Cloud Armor | 글로벌 Anycast, ML 적응형 |
| **Cloudflare** | Magic Transit / DDoS | 최대 195Tbps 흡수 용량 |
| **Naver Cloud** | DDoS Protection | 국내 트래픽 특화, 스크러빙 |
| **Akamai** | Prolexic | 20Tbps 스크러빙 용량 |

---

## 6. SIEM (Security Information and Event Management)

### 역할

```
모든 보안 장비·서버·애플리케이션의 로그를 중앙 수집 후
  1. 정규화 (서로 다른 포맷 통일)
  2. 상관 분석 (여러 이벤트를 연결해 공격 흐름 파악)
  3. 알림 (임계치·룰 기반)
  4. 포렌식 지원 (사후 조사)

예시: 상관 분석으로 발견하는 공격
  10:00  WAF: SQLi 시도 차단 (192.168.1.100)
  10:01  IPS: 포트 스캔 탐지 (192.168.1.100)
  10:02  서버 로그: 로그인 실패 50회 (192.168.1.100)
  10:03  서버 로그: 로그인 성공 (192.168.1.100)
  → SIEM: 브루트포스 후 성공 → 즉시 알림 + IP 차단
```

### 핵심 기능

```
[수집 (Collection)]
  Syslog, API, Agent 방식으로 로그 수집
  AWS CloudTrail, VPC Flow Logs, WAF Logs, OS Logs

[정규화 (Normalization)]
  각기 다른 포맷 → 공통 스키마로 변환
  ECS (Elastic Common Schema), CEF (Common Event Format)

[상관 분석 (Correlation)]
  시간·IP·사용자·이벤트 타입 기반 패턴 매칭
  예: 동일 IP에서 30초 내 100회 실패 → 브루트포스

[알림 (Alerting)]
  임계치 초과 → Slack/PagerDuty/이메일

[대응 (SOAR 통합)]
  Security Orchestration, Automation and Response
  → 알림 수신 시 자동으로 IP 차단, 티켓 생성, 격리 실행
```

| 벤더 | 상품명 | 특징 |
|---|---|---|
| **AWS** | Security Hub | 여러 AWS 서비스 결과 통합, SIEM-lite |
| **AWS** | Amazon Detective | 시각화 기반 조사, GuardDuty 연동 |
| **Azure** | Microsoft Sentinel | 클라우드 네이티브 SIEM+SOAR, KQL 쿼리 |
| **GCP** | Chronicle SIEM | 구글 인프라 기반, 무제한 로그 보관 |
| **Elastic** | Elastic SIEM | ELK 스택 기반, 오픈소스 |
| **Splunk** | Splunk Enterprise Security | 엔터프라이즈 표준, 강력한 SPL 쿼리 |
| **IBM** | QRadar | 전통 SIEM 강자, 온프레미스·클라우드 |

---

## 7. CSPM (Cloud Security Posture Management)

```
역할: 클라우드 설정 오류·보안 취약점을 자동으로 탐지
     "S3 버킷 퍼블릭 공개됨", "보안그룹 0.0.0.0/0 허용" 등

예시 탐지 항목:
  · S3/Blob 버킷 퍼블릭 노출
  · 루트 계정 MFA 미설정
  · 과도한 IAM 권한 (FullAccess 남용)
  · 암호화 미적용 볼륨/DB
  · 보안그룹 0.0.0.0:22 오픈
  · CloudTrail 로깅 비활성화
  · 기본 VPC 사용

컴플라이언스 프레임워크 매핑:
  CIS AWS Benchmark, NIST CSF, ISO 27001, PCI-DSS, ISMS-P
```

| 벤더 | 상품명 | 특징 |
|---|---|---|
| **AWS** | Security Hub + Config | AWS 리소스 보안 점검, Config Rules |
| **Azure** | Defender for Cloud | Secure Score, 권고사항 자동 생성 |
| **GCP** | Security Command Center | 자산 인벤토리 + 취약점 탐지 |
| **Wiz** | Cloud Security Platform | 멀티클라우드 CSPM 선두, 그래프 기반 |
| **Prisma Cloud** | (Palo Alto) | CSPM + CWPP + CIEM 통합 |
| **Orca Security** | SideScanning™ | 에이전트 없이 스냅샷 기반 스캔 |

---

## 8. 기타 주요 보안 상품

### Secrets Manager / KMS

```
역할: 비밀키·자격증명·API 키를 안전하게 저장·배포·교체

문제 상황: 코드에 DB 비밀번호 하드코딩 → GitHub 노출 → 해킹
해결: Secrets Manager에 저장, 런타임에 API로 조회

기능:
  · 비밀값 버전 관리
  · 자동 교체 (RDS 비밀번호 주기적 로테이션)
  · IAM 기반 접근 제어
  · 감사 로그

상품: AWS Secrets Manager, Azure Key Vault, GCP Secret Manager
     HashiCorp Vault (멀티클라우드)
```

### Certificate Manager (ACM/ACME)

```
역할: TLS/SSL 인증서 발급·갱신·배포 자동화

기능:
  · 무료 인증서 발급 (도메인 소유 검증)
  · 자동 갱신 (만료 전 자동 재발급)
  · LB/CDN 자동 배포

상품: AWS ACM, Azure App Service Certificate
     Let's Encrypt + Certbot (자체 관리)
```

### CASB (Cloud Access Security Broker)

```
역할: 사용자 ↔ 클라우드 서비스 사이에 위치
     비인가 SaaS 사용 탐지·제어 (Shadow IT)

기능:
  · Shadow IT 탐지 (비인가 Dropbox, 개인 구글 드라이브 사용)
  · 클라우드 앱 접근 제어
  · DLP (Data Loss Prevention): 민감 데이터 업로드 차단
  · 이상 사용 탐지 (새벽 3시 대량 다운로드)

상품: Microsoft Defender for Cloud Apps, Netskope, Zscaler CASB
```

### Zero Trust / ZTNA

```
개념: "네트워크 위치를 신뢰하지 않는다"
     내부망에 있어도 매번 인증·인가 확인

기존 VPN 방식:
  VPN 접속 → 내부망 전체 접근 가능
  → VPN 계정 탈취 시 모든 내부 자원 노출

ZTNA 방식:
  접속 시도 → 사용자 신원 + 디바이스 상태 + 컨텍스트 확인
  → 요청한 특정 애플리케이션만 접근 허용 (최소권한)

상품: Cloudflare Access, Zscaler ZPA, BeyondCorp (Google)
     AWS Verified Access
```

---

## 9. 공공·금융 환경 적용 요약

| 보안 요구사항 | 해당 상품 | 법령 근거 |
|---|---|---|
| 인터넷 공격 차단 | WAF | 국가 사이버보안 기본지침, ISMS-P 2.6.1 |
| 침입 탐지·차단 | IPS/IDS | 국가사이버안전관리규정 제21조 |
| 악성코드 방지 | AV/EDR | ISMS-P 2.9.3 |
| DDoS 대응 | DDoS Protection | 국가 사이버보안 기본지침 |
| 통합 로그·감사 | SIEM | ISMS-P 2.9.4, 개인정보보호법 제29조 |
| 클라우드 설정 점검 | CSPM | CSAP 인증 요구사항 |
| 비밀키 관리 | Secrets Manager / KMS | 정보통신망법 제45조 |
| 네트워크 분리 | 방화벽 + IPS | 국가사이버안전관리규정 제17조, N2SF |

---

## 10. 상품 선택 흐름 — 규모별 권장 구성

```
[소규모 — 스타트업, 소형 서비스]
  필수: WAF (Cloudflare Free/Pro or AWS WAF)
        DDoS Protection (Cloudflare 또는 AWS Shield Standard)
        Secrets Manager (비밀키 코드 하드코딩 제거)
  권장: CSPM (AWS Security Hub 무료 티어)

[중규모 — 엔터프라이즈, 공공기관]
  필수: WAF + IPS + DDoS Protection
        AV / EDR (서버 에이전트)
        SIEM (로그 수집·알림)
        Secrets Manager + KMS
        CSPM (정기 설정 점검)
  권장: CASB (SaaS 제어), ZTNA (VPN 대체)

[대규모 — 금융, 공공 클라우드 CSAP 상]
  위 전체 +
  WAF Custom 룰 운영팀
  24/7 SOC (Security Operations Center)
  SOAR 자동 대응
  취약점 스캐너 (Tenable, Qualys)
  레드팀·침투 테스트 정기 수행
```

---

## 11. 관련
- [[../../programming/security/Web-Attacks]] · [[../../programming/security/Network-Attacks]]
- [[../../programming/security/Cryptography]]
- [[../architecture/Public-Network-Security]]
