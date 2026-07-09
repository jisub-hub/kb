---
tags:
  - infra
  - architecture
  - 3-tier
  - ha
created: 2026-06-15
---

# 3-Tier Architecture

> [!summary] 한 줄 요약
> **Web(프레젠테이션) / WAS(비즈니스 로직) / DB(데이터)** 3계층으로 역할을 분리하는 전통적 인프라 아키텍처. 각 계층을 독립적으로 확장·교체할 수 있다.

---

## 1. 계층 구조

```
인터넷
  ↓
[Load Balancer]            ← Public Subnet, 단일 진입점
  ↓
[Web Tier]                 ← Public/DMZ Subnet
  Nginx / Web Server       ← 정적 콘텐츠, SSL Termination, 리버스 프록시
  ↓
[WAS Tier]                 ← Private Subnet
  Spring Boot / App Server ← 비즈니스 로직, API
  ↓
[DB Tier]                  ← DB Subnet (격리)
  PostgreSQL / MySQL       ← Primary + Standby
```

---

## 2. 역할별 계층 분리 이유

| 계층 | 역할 | 분리 이유 |
|---|---|---|
| **Web** | 정적 파일 서빙, SSL 처리, 리버스 프록시 | Nginx 최적화 따로, WAS 언어/프레임워크 독립 |
| **WAS** | 비즈니스 로직, DB 연결, 세션 관리 | 독립 스케일 아웃 가능 |
| **DB** | 영구 데이터 저장 | 보안 격리, 독립 스케일 업 |

---

## 3. 이중화(HA) 설계

```
인터넷
  ↓
[L7 Load Balancer] (클라우드 관리형 — HA 기본 제공)
    ├→ [Web-1] (AZ-A, Public SN)
    └→ [Web-2] (AZ-B, Public SN)
          ↓  (Cross-Connected)
    ├→ [WAS-1] (AZ-A, Private SN)
    └→ [WAS-2] (AZ-B, Private SN)
          ↓
    ┌→ [DB Primary] (AZ-A, DB SN)
    └→ [DB Standby] (AZ-B, DB SN) ← 동기 복제
```

**Cross-Connected 이중화 — 필수:**  
→ [[../network/HA-Architecture]] 참고

---

## 4. 서브넷 설계 (클라우드 기준)

```
VCN: 10.0.0.0/16

Public Subnet:  10.0.1.0/24  → LB, Bastion Host
                             Route: 0.0.0.0/0 → IGW

Web Subnet:     10.0.2.0/24  → Web Server (Nginx)
                             Route: 0.0.0.0/0 → IGW (공인IP 없으면 NAT GW)

WAS Subnet:     10.0.3.0/24  → Spring Boot WAS
                             Route: 0.0.0.0/0 → NAT GW

DB Subnet:      10.0.4.0/24  → DB (인터넷 연결 없음)
                             Route: 내부만
```

---

## 5. 보안 ACG 설계

```
LB ACG:
  Inbound:  80, 443 ← 0.0.0.0/0
  Outbound: 80 → Web Subnet

Web ACG:
  Inbound:  80 ← LB Subnet
  Outbound: 8080 → WAS Subnet

WAS ACG:
  Inbound:  8080 ← Web Subnet
  Outbound: 5432 → DB Subnet
            443 → 0.0.0.0/0 (외부 API 호출)

DB ACG:
  Inbound:  5432 ← WAS Subnet
  Outbound: (최소화)
```

---

## 6. 배포 흐름 (Blue-Green 예시)

```
1. WAS-v2 신규 인스턴스 생성 (WAS Subnet)
2. LB에서 WAS-v2로 트래픽 전환 (10% Canary)
3. 정상 확인 후 100% 전환
4. WAS-v1 종료
```

---

## 7. 3-Tier의 한계 → MSA로 진화

| 한계 | 내용 |
|---|---|
| 기술 스택 종속 | 모든 WAS가 같은 언어/프레임워크 |
| 스케일 단위 | WAS 전체를 스케일 아웃 (기능별 독립 불가) |
| 배포 단위 | 전체 WAS 재배포 (기능 하나 바꿔도) |
| 장애 격리 | WAS 하나 장애가 전체 서비스 영향 |

→ 기능별 독립 배포·확장이 필요해지면 [[MSA-Architecture]] 고려

---

## 8. 관련
- [[../network/HA-Architecture]] · [[../network/Security-ACG-NSG]] · [[../network/VCN-VPC]] · [[MSA-Architecture]]
