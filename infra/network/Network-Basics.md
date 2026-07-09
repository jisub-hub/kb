---
tags:
  - network
  - ip
  - osi
  - tcp-ip
created: 2026-06-15
---

# 네트워크 기초 — IP 주소 & 계층 모델

> [!summary] 한 줄 요약
> 클라우드 보안 규칙(ACG/NSG)은 **L3(IP) + L4(포트)** 기준으로 동작한다. IP 주소 체계와 계층 모델을 이해해야 방화벽 룰을 올바르게 설계할 수 있다.

---

## 1. IP 주소 체계

### IPv4 구조
- **32bit** 주소. 8bit 4개 옥텟으로 표현 → `192.168.1.10`
- 각 옥텟 범위: 0 ~ 255

### Public IP vs Private IP

| | Public IP | Private IP |
|---|---|---|
| 범위 | 전 세계 유일 | RFC 1918 사설 대역만 |
| 인터넷 접근 | 직접 가능 | IGW / NAT 경유 필요 |
| 클라우드 용도 | Load Balancer, Bastion Host | VM, DB, WAS |

**RFC 1918 사설 대역 (암기 권장)**
```
10.0.0.0/8       → 대형 VCN/VPC 전체 대역으로 자주 사용
172.16.0.0/12    → 중형
192.168.0.0/16   → 소형 / 로컬 환경
```

---

## 2. OSI 7계층 — 개발자 관련 3계층

전체 7계층 중 **인프라·보안 설계에서 직접 다루는 계층**은 3개다.

| 계층 | 이름 | 역할 | 관련 기술 |
|---|---|---|---|
| **L3** | 네트워크층 | IP 기반 라우팅 | IP, ICMP, 라우팅 테이블 |
| **L4** | 전송층 | 포트 기반 연결 | TCP, UDP, ACG/NSG 룰 |
| **L7** | 응용층 | 애플리케이션 프로토콜 | HTTP, HTTPS, DNS, SSH |

> [!tip] 보안 규칙 설계 시
> - ACG / NSG → **L3(소스 IP) + L4(포트/프로토콜)** 조합으로 허용/차단
> - API Gateway / ALB → **L7(URL path, Host header)** 기반 라우팅까지 가능

---

## 3. TCP vs UDP

| | TCP | UDP |
|---|---|---|
| 연결 방식 | 3-way handshake (연결형) | 비연결형 |
| 신뢰성 | 순서 보장, 재전송 | 보장 없음 |
| 속도 | 상대적으로 느림 | 빠름 |
| 용도 | HTTP, SSH, DB 연결 | DNS, 스트리밍, 게임 |
| 클라우드 ACG 설정 | 대부분 TCP | DNS(53) 등 UDP 별도 허용 |

---

## 4. 주요 포트 번호 (보안 룰 설계 기준)

| 포트 | 프로토콜 | 용도 | ACG 설정 시 주의 |
|---|---|---|---|
| 22 | TCP | SSH | **특정 IP만** 허용. 0.0.0.0/0 절대 금지 |
| 80 | TCP | HTTP | 웹서버 인바운드, 보통 443으로 리다이렉트 |
| 443 | TCP | HTTPS | 웹서버 인바운드 |
| 3306 | TCP | MySQL | WAS 서브넷 IP로만 제한 |
| 5432 | TCP | PostgreSQL | WAS 서브넷 IP로만 제한 |
| 6379 | TCP | Redis | 내부망만 허용 |
| 8080/8443 | TCP | WAS (Tomcat) | Web 서브넷에서만 허용 |

---

## 5. DNS 기초

- 도메인(`api.example.com`) → IP 주소 변환
- UDP 53번 포트 사용 (대용량 응답은 TCP도 사용)
- 클라우드에서 **내부 DNS**: VM 간 Private IP를 도메인으로 접근 가능 (OCI: 기본 제공, AWS: Route53 Private Hosted Zone)

---

## 6. 관련
- [[Subnetting]] · [[VCN-VPC]] · [[Security-ACG-NSG]]
