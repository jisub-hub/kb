---
tags:
  - network
  - bgp
  - anycast
  - routing
  - ddos
created: 2026-06-16
---

# BGP & Anycast — 라우팅 프로토콜과 분산 IP

> [!summary] 한 줄 요약
> **BGP(Border Gateway Protocol)**는 인터넷 자율 시스템(AS) 간 경로를 교환하는 **사실상 유일한 외부 라우팅 프로토콜**. **Anycast**는 동일 IP를 여러 위치에 광고해 **가장 가까운 노드로 자동 라우팅**하는 기법. DDoS 방어와 DNS/CDN의 핵심.

---

## 1. BGP 기초 개념

### 자율 시스템 (AS)

```
[ISP A - AS1234]  ←BGP세션→  [ISP B - AS5678]  ←BGP세션→  [ISP C - AS9999]
    │                                                              │
[내 회사 AS3456] ←──────────────────────────────────────────────→ [CDN AS2222]
```

- **AS (Autonomous System)**: 단일 라우팅 정책 하에 관리되는 IP 대역 집합
- **ASN (AS Number)**: IANA 할당 고유 번호 (16비트/32비트)
- **BGP**: AS 간 경로(prefix) 광고 프로토콜. TCP 179번 포트
- **NLRI (Network Layer Reachability Information)**: 광고하는 IP prefix (`203.0.113.0/24`)

### eBGP vs iBGP

| | eBGP | iBGP |
|--|------|------|
| 사용처 | 서로 다른 AS 간 | 같은 AS 내 라우터 간 |
| TTL | 1 (직접 연결) | 255 (내부) |
| 경로 변경 | next-hop 자신으로 변경 | next-hop 그대로 유지 |

---

## 2. BGP 경로 속성 & 정책

```
# BGP 경로 선택 순서 (높을수록 우선)
1. Weight (Cisco 전용, 로컬)
2. LOCAL_PREF (높을수록 선호 — AS 내 정책)
3. Locally Originated (자신이 광고한 경로)
4. AS_PATH 길이 (짧을수록 선호)
5. ORIGIN (IGP > EGP > ?)
6. MED (낮을수록 선호 — 상대 AS에 힌트)
7. eBGP > iBGP
8. IGP metric (next-hop까지 거리)
```

```bash
# 실제 BGP 테이블 조회 (FRRouting/Quagga)
vtysh
  show bgp summary                   # 피어 상태
  show bgp ipv4 unicast              # 전체 BGP 테이블
  show bgp ipv4 unicast 203.0.113.0/24  # 특정 prefix 경로
  show ip route bgp                  # BGP로 학습된 경로만
```

---

## 3. Anycast — 동일 IP 분산 배치

```
[서울 - 203.0.113.1/32 광고]
        │
[BGP 인터넷 코어]  →  사용자는 가장 가까운 PoP으로 자동 라우팅
        │
[뉴욕 - 203.0.113.1/32 광고]   ← 동일 IP를 다른 AS 위치에서 광고
[프랑크푸르트 - 203.0.113.1/32 광고]
```

### Anycast 용도

| 용도 | 예시 |
|------|------|
| **DNS** | `1.1.1.1` (Cloudflare), `8.8.8.8` (Google) — 전 세계 동일 IP |
| **CDN** | 정적 콘텐츠를 가장 가까운 PoP에서 서빙 |
| **DDoS 방어** | 공격 트래픽을 스크러빙 센터로 흡수 |
| **글로벌 LB** | 지역 기반 트래픽 분산 |

### Anycast DDoS 방어 원리

```
[DDoS 공격 100Gbps]
        │
[BGP Anycast → 공격 트래픽이 여러 PoP으로 분산]
  서울 PoP: 25Gbps  → 스크러빙 → 정상 트래픽 Origin 전달
  뉴욕 PoP: 30Gbps  → 스크러빙 ↑
  런던 PoP: 25Gbps  → 스크러빙 ↑
  도쿄 PoP: 20Gbps  → 스크러빙 ↑
        │
    [Origin 서버]: 공격 없이 정상 운영
```

---

## 4. BGP Blackhole (긴급 DDoS 차단)

```bash
# RTBH (Remotely Triggered Blackhole): 공격 목표 IP를 BGP로 null 라우팅
# 대역폭은 소비하지만 서버까지 트래픽 도달 차단

# 자체 BGP 라우터에서 (FRRouting)
ip route 203.0.113.50/32 Null0    # 로컬 블랙홀
# BGP community 66000:666 으로 상위 ISP에 블랙홀 요청

# Cloudflare Magic Transit / AWS Shield Advanced
# → 상위 AS에서 공격 IP 자동 광고 차단
```

---

## 5. 공공/기업 BGP 실무

### AS 취득이 필요한 경우
- 멀티홈(두 ISP 이상에 연결)
- IP 주소 이동성 필요 (ISP 변경 시 IP 유지)
- 글로벌 Anycast 운영

### 현실적 대안 (AS 없이)
- **Cloudflare / Akamai**: BGP/Anycast를 서비스로 제공
- **AWS Global Accelerator**: Anycast IP + AWS 백본 라우팅
- **GCP Cloud CDN / Azure Front Door**: 동일 패턴

```bash
# BGP 상태 공개 조회 도구
# BGP Looking Glass: route-views.oregon-ix.net
telnet route-views.oregon-ix.net
  show ip bgp 203.0.113.0/24

# BGP 커뮤니티 정보
# bgp.he.net — ASN 조회, BGP 경로 시각화
```

---

## 6. 관련
- [[Network-Attacks]] — DDoS 방어 아키텍처에서 Anycast 언급
- [[Load-Balancer]] · [[VPN]] · [[Network-Basics]]
- [[../security/Cloud-Security-Products]] — Anycast 기반 DDoS 상품
