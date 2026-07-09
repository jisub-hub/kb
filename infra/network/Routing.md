---
tags:
  - network
  - routing
  - routing-table
  - port-forwarding
  - hop
created: 2026-06-16
---

# 라우팅 — Routing Table · Hop · Port Forwarding

> [!summary] 한 줄 요약
> **라우팅**은 패킷이 목적지까지 가는 경로를 결정하는 과정. 라우터는 라우팅 테이블을 보고 다음 홉(Next Hop)으로 패킷을 전달한다. **Port Forwarding**은 외부 포트를 내부 IP:Port로 매핑하는 NAT 기법.

---

## 1. 라우팅 동작 원리

```
[패킷 전달 과정]

  PC (192.168.1.10)
    │
    │ 목적지: 8.8.8.8 (Google DNS)
    │ 라우팅 테이블 확인: "내 서브넷(192.168.1.0/24) 아님 → 기본 게이트웨이로"
    ▼
  Router-1 (192.168.1.1)     ← Hop 1
    │
    │ 라우팅 테이블 확인: "인터넷 경로 → ISP 라우터로"
    ▼
  ISP Router                 ← Hop 2
    │
    │ BGP 라우팅: "8.8.8.0/24 → Google 네트워크 방향"
    ▼
  Google Router              ← Hop 3
    │
    ▼
  8.8.8.8 (목적지 도착)

[핵심 개념]
  라우터: 패킷을 받아 목적지 IP를 라우팅 테이블과 비교, 다음 라우터로 전달
  홉(Hop): 라우터를 거칠 때마다 +1. TTL이 0이 되면 패킷 폐기
  TTL: IP 헤더 필드, 무한 루프 방지 (기본값: Linux=64, Windows=128)
```

---

## 2. 라우팅 테이블 (Routing Table)

라우터·서버 OS가 가진 **목적지 → 출구 인터페이스 또는 다음 홉** 매핑 테이블.

```bash
# Linux 라우팅 테이블 확인
ip route show
# 또는
route -n

# 출력 예:
# Destination     Gateway         Genmask         Flags  Metric Iface
# 0.0.0.0         192.168.1.1     0.0.0.0         UG     100    eth0  ← 기본 라우트
# 192.168.1.0     0.0.0.0         255.255.255.0   U      0      eth0  ← 직접 연결 서브넷
# 10.0.0.0        10.0.0.1        255.0.0.0       UG     0      tun0  ← VPN 경로
```

### 라우팅 테이블 항목 해석

```
[컬럼 설명]
  Destination: 목적지 네트워크 (0.0.0.0 = 기본 라우트, 모든 목적지 매치)
  Gateway:     다음 홉 IP (0.0.0.0이면 직접 연결, 라우터 없이 도달 가능)
  Genmask:     서브넷 마스크 (255.255.255.0 = /24)
  Flags:       U=Up, G=Gateway(라우터 경유 필요), H=Host
  Metric:      경로 우선순위 (낮을수록 우선)
  Iface:       사용할 네트워크 인터페이스 (eth0, tun0, ...)

[라우팅 결정 알고리즘: Longest Prefix Match]
  목적지 IP: 192.168.1.50
  
  후보 경로:
    0.0.0.0/0    → 기본 라우트 (0비트 일치)
    192.168.1.0/24 → /24 일치 (24비트 일치)  ← 선택 (더 구체적)
  
  → 가장 긴(구체적인) 프리픽스 매치를 선택
```

### 라우팅 테이블 수동 조작

```bash
# 경로 추가
ip route add 10.10.0.0/24 via 192.168.1.1
# 또는 특정 인터페이스로
ip route add 10.20.0.0/24 dev eth1

# 기본 게이트웨이 변경
ip route replace default via 192.168.1.254

# 경로 삭제
ip route del 10.10.0.0/24

# 영구 저장 (Ubuntu/Debian: /etc/netplan/...)
# /etc/netplan/01-config.yaml에 routes 섹션 추가
```

---

## 3. 홉(Hop) 상세

```
[TTL과 홉]
  IP 패킷 헤더에 TTL(Time To Live) 필드 존재
  라우터를 통과할 때마다 TTL - 1
  TTL = 0이 되면 라우터가 패킷 폐기 + ICMP "Time Exceeded" 오류 전송
  
  목적: 라우팅 루프 발생 시 패킷이 영원히 돌지 않도록

[traceroute / tracert]
  각 홉에서 TTL을 1부터 늘려가며 경로 추적
```

```bash
# 경로 추적 (Linux)
traceroute 8.8.8.8

# 결과 예:
# 1  192.168.1.1 (게이트웨이)         1.2 ms
# 2  10.0.0.1 (ISP 라우터)          8.4 ms
# 3  203.x.x.x  (백본)              15.1 ms
# 4  8.8.8.8  (목적지)              12.8 ms

# ICMP 대신 TCP 사용 (방화벽 우회)
traceroute -T -p 443 8.8.8.8

# Windows
tracert 8.8.8.8

# MTR: 실시간 경로 모니터링
mtr 8.8.8.8
```

---

## 4. Port Forwarding (포트 포워딩)

외부 IP:Port로 들어오는 트래픽을 내부 IP:Port로 **NAT(Network Address Translation)** 으로 전달.

```
[기본 구조]

  인터넷
    │
    │ 요청: 203.x.x.x:8080 (공인 IP)
    ▼
  공유기/NAT Router (203.x.x.x 공인 IP)
    │
    │ Port Forwarding 규칙: :8080 → 192.168.1.100:8080
    ▼
  내부 서버 (192.168.1.100:8080)

[용도]
  - 홈 서버를 인터넷에서 접근 가능하게
  - 클라우드 VM의 다른 포트로 트래픽 리다이렉트
  - Docker 컨테이너 포트 노출 (docker -p 8080:80)
  - SSH 터널링을 통한 원격 서비스 접근
```

### Linux iptables 포트 포워딩

```bash
# IP 포워딩 활성화
echo 1 > /proc/sys/net/ipv4/ip_forward
# 영구 설정: /etc/sysctl.conf에 net.ipv4.ip_forward=1

# 포트 포워딩: 외부 8080 → 내부 192.168.1.100:80
iptables -t nat -A PREROUTING \
  -p tcp --dport 8080 \
  -j DNAT --to-destination 192.168.1.100:80

# 마스커레이드 (NAT 응답 처리)
iptables -t nat -A POSTROUTING \
  -j MASQUERADE

# 포워딩 허용
iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 80 -j ACCEPT
```

### SSH 포트 포워딩 (터널링)

```bash
# 로컬 포워딩: 로컬 9200 → 원격 서버를 통해 ES 9200 접근
# (Elasticsearch가 방화벽 안쪽에 있을 때)
ssh -L 9200:elasticsearch-host:9200 user@bastion-host
# → localhost:9200으로 ES 접근 가능

# 원격 포워딩: 원격 서버 8080 → 내 로컬 8080
# (로컬 개발 서버를 외부에서 테스트할 때)
ssh -R 8080:localhost:8080 user@public-server

# SOCKS 프록시 (동적 포워딩)
ssh -D 1080 user@bastion-host
# → localhost:1080을 SOCKS5 프록시로 사용 (모든 트래픽 터널)
```

---

## 5. 라우팅 프로토콜

```
[정적 라우팅 vs 동적 라우팅]

정적 라우팅:
  - 관리자가 직접 라우팅 테이블에 경로 추가
  - 소규모 네트워크, 예측 가능한 트래픽
  - 변경 시 수동 업데이트 필요

동적 라우팅:
  - 라우터끼리 라우팅 정보 교환, 자동으로 최적 경로 계산
  - IGP (내부 게이트웨이): OSPF, EIGRP
  - EGP (외부 게이트웨이): BGP (인터넷 백본)

[클라우드에서의 라우팅]
  VPC 내부: VPC 라우팅 테이블 (서브넷별 설정)
  인터넷 접근: IGW(Internet Gateway) + 라우팅 테이블에 0.0.0.0/0 → IGW
  내부 통신: 서브넷 간 라우팅은 기본 허용
  VPN: VGW(Virtual Private Gateway)로 온프레미스 경로 추가
  VPC Peering: 다른 VPC 대역을 라우팅 테이블에 추가
```

---

## 6. 클라우드 라우팅 테이블 (AWS VPC 예시)

```
[Public 서브넷 라우팅 테이블]
  Destination     Target
  10.0.0.0/16     local       ← VPC 내부 통신 (자동)
  0.0.0.0/0       igw-xxx     ← 인터넷 접근 (IGW)

[Private 서브넷 라우팅 테이블]
  Destination     Target
  10.0.0.0/16     local       ← VPC 내부
  0.0.0.0/0       nat-xxx     ← 아웃바운드만 (NAT Gateway)
  10.0.0.0/8      vgw-xxx     ← VPN을 통해 온프레미스 접근

[Transit Gateway (멀티 VPC)]
  여러 VPC + 온프레미스를 중앙 허브로 연결
  각 VPC의 라우팅 테이블에 TGW 추가
```

---

## 7. 관련
- [[Network-Basics]] — OSI 계층, IP 주소, TCP/UDP 기초
- [[VCN-VPC]] — 클라우드 가상 네트워크
- [[Gateway]] — IGW, NAT GW, VPN GW
- [[BGP-Anycast]] — BGP 프로토콜 상세
- [[../linux/TCPDump-Wireshark]] — 패킷 캡처로 라우팅 경로 디버깅
