---
tags:
  - linux
  - network
  - tcpdump
  - wireshark
  - troubleshooting
  - packet-capture
created: 2026-06-16
---

# TCPDump & Wireshark — 네트워크 패킷 분석

> [!summary] 한 줄 요약
> **TCPDump**는 서버에서 실시간 패킷 캡처·필터링하는 CLI 도구, **Wireshark**는 GUI로 심층 분석하는 도구. "연결은 되는데 데이터가 안 온다", "간헐적 응답 지연", "TLS 핸드셰이크 실패" 등 **로그로 보이지 않는 네트워크 문제**의 최후 수단.

---

## 1. 언제 TCPDump/Wireshark를 써야 하는가

```
[로그로 해결 가능 → TCPDump 불필요]
  - 500 에러: 애플리케이션 로그에 stacktrace 있음
  - 인증 실패: Authorization 헤더 로그 확인
  - DB 쿼리 느림: slow query log 확인

[TCPDump가 필요한 상황]
  ✅ "TCP connection refused" — 포트 열려 있는데 연결 안 됨
  ✅ "Connection reset by peer" — 누가 TCP RST 보내는지 불명
  ✅ 간헐적 패킷 손실 — 재전송(retransmission) 발생 여부 확인
  ✅ TLS 핸드셰이크 실패 — Client Hello/Server Hello 교환 확인
  ✅ 로드밸런서 health check 비정상 — 실제 패킷 수준 확인
  ✅ 방화벽 규칙 검증 — 패킷이 실제로 도착하는지 확인
  ✅ API 타임아웃 — TCP 3-way handshake 완료 후 응답 없는지 확인
  ✅ DNS 해석 문제 — DNS 쿼리/응답 패킷 직접 확인
  ✅ 프로토콜 구현 디버깅 — MQTT, RTSP, 커스텀 바이너리 프로토콜
```

---

## 2. TCPDump 기본 사용법

### 인터페이스 및 기본 옵션

```bash
# 사용 가능한 인터페이스 목록
tcpdump -D
# 1.eth0
# 2.lo
# 3.docker0
# 4.any (모든 인터페이스)

# 기본 캡처 (eth0)
tcpdump -i eth0

# 모든 인터페이스
tcpdump -i any

# 주요 옵션
# -n   : IP 주소 역방향 DNS 해석 안 함 (빠름)
# -nn  : 포트도 서비스 이름으로 변환 안 함 (80→http 변환 없음)
# -v   : 상세 정보 (TTL, DF bit 등)
# -vv  : 더 상세
# -X   : 패킷 내용 HEX + ASCII 출력
# -A   : 패킷 내용 ASCII만 출력 (HTTP 내용 확인 시)
# -s0  : 패킷 전체 캡처 (기본 65535, 0=전체)
# -c N : N개 패킷 캡처 후 종료
# -w   : pcap 파일로 저장
# -r   : pcap 파일 읽기
```

### 필터 표현식 (BPF — Berkeley Packet Filter)

```bash
# ── 호스트 필터 ──────────────────────────────────────────────────
# 특정 IP 트래픽
tcpdump -i eth0 -nn host 192.168.1.100

# 특정 IP 발신
tcpdump -i eth0 -nn src host 192.168.1.100

# 특정 IP 수신
tcpdump -i eth0 -nn dst host 192.168.1.100

# 특정 네트워크
tcpdump -i eth0 -nn net 192.168.1.0/24

# ── 포트 필터 ───────────────────────────────────────────────────
# 특정 포트
tcpdump -i eth0 -nn port 8080

# 특정 포트 범위
tcpdump -i eth0 -nn portrange 8080-8090

# 발신 포트
tcpdump -i eth0 -nn src port 443

# ── 프로토콜 필터 ────────────────────────────────────────────────
tcpdump -i eth0 -nn tcp
tcpdump -i eth0 -nn udp
tcpdump -i eth0 -nn icmp

# ── 조합 필터 (and, or, not) ─────────────────────────────────────
# 특정 IP의 8080 포트
tcpdump -i eth0 -nn "host 192.168.1.100 and port 8080"

# DB 서버 포트 제외 (노이즈 제거)
tcpdump -i eth0 -nn "not port 5432 and not port 6379"

# SYN 패킷만 (새 연결 시도)
tcpdump -i eth0 -nn "tcp[tcpflags] & tcp-syn != 0"

# RST 패킷만 (비정상 연결 종료)
tcpdump -i eth0 -nn "tcp[tcpflags] & tcp-rst != 0"

# HTTP GET 요청만
tcpdump -i eth0 -A "port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420"
```

---

## 3. 시나리오별 TCPDump 명령

### 3.1 TCP 연결 문제 (Connection refused / reset)

```bash
# 목표: 서버가 SYN을 받는지, RST을 보내는지 확인
tcpdump -i eth0 -nn "host 10.0.0.5 and port 8080" -v

# 정상 연결:
#   클라이언트 → SYN
#   서버 → SYN-ACK
#   클라이언트 → ACK
#   (데이터 교환)
#   FIN / FIN-ACK

# Connection refused:
#   클라이언트 → SYN
#   서버 → RST, ACK   ← 포트가 닫혀 있음 (서버 프로세스가 없거나 방화벽)

# Connection timeout:
#   클라이언트 → SYN
#   (응답 없음, 재전송 SYN 3회)   ← 방화벽 DROP, 라우팅 문제

# RST 발신자 특정
tcpdump -i any -nn "tcp[tcpflags] & tcp-rst != 0 and host 10.0.0.5"
```

### 3.2 HTTP/HTTPS 트래픽 확인

```bash
# HTTP 요청/응답 내용 (암호화 없는 경우)
tcpdump -i eth0 -A -nn "port 80" | grep -E "^(GET|POST|PUT|DELETE|HTTP)"

# HTTP 헤더 + 바디 전체
tcpdump -i eth0 -A -s0 "port 80"

# Spring Boot Actuator health check 트래픽 확인
tcpdump -i lo -A -nn "port 8080" | grep -A2 "/actuator/health"

# TLS Client Hello 확인 (SNI 도메인 추출)
tcpdump -i eth0 -nn "port 443 and tcp[((tcp[12:1] & 0xf0) >> 2):1] = 22" -X | \
  grep -A2 "Server Name"
```

### 3.3 DNS 문제

```bash
# DNS 쿼리/응답 확인
tcpdump -i eth0 -nn "udp port 53" -v

# 출력 예시:
# 10.0.0.1.52341 > 8.8.8.8.53: A? api.example.com
# 8.8.8.8.53 > 10.0.0.1.52341: A api.example.com [3 answers]
#   A 203.0.113.1 TTL 60
#   A 203.0.113.2 TTL 60

# DNS NXDOMAIN (도메인 없음)
tcpdump -i eth0 -nn "udp port 53" | grep "NXDOMAIN\|NXDomain"

# 특정 도메인 쿼리 추적
tcpdump -i eth0 -nn -v "udp port 53" | grep "api.example.com"
```

### 3.4 MQTT / 커스텀 프로토콜

```bash
# MQTT 브로커(1883) 트래픽 캡처
tcpdump -i eth0 -nn -X "port 1883" -w /tmp/mqtt-capture.pcap

# → Wireshark에서 열어 MQTT 프로토콜 디코딩 (Wireshark가 자동 인식)

# 특정 MQTT 브로커와의 연결만
tcpdump -i eth0 -nn "host 10.0.0.10 and port 1883"

# RTSP 스트리밍 트래픽
tcpdump -i eth0 -nn "port 554" -v

# TCP 소켓 커스텀 프로토콜 (포트 9001)
tcpdump -i eth0 -nn -X "port 9001" | head -100
```

### 3.5 패킷 손실 / 재전송 감지

```bash
# TCP 재전송 패킷 캡처 (파일로 저장 후 Wireshark 분석)
tcpdump -i eth0 -nn -w /tmp/retransmit-$(date +%Y%m%d_%H%M%S).pcap \
  "host 10.0.0.5 and port 8080"

# Wireshark 필터: tcp.analysis.retransmission
# → 얼마나 자주 재전송이 발생하는지 확인

# 빠른 현장 확인: ss + netstat
ss -s    # 전체 소켓 통계
netstat -s | grep -i retransmit
```

### 3.6 파일로 저장 → Wireshark 분석

```bash
# 서버에서 캡처 → 로컬로 전송 → Wireshark 분석
# (서버에 GUI 없는 경우)

# 1. 캡처 시작 (최대 100MB 또는 1000개 패킷)
tcpdump -i eth0 -nn -s0 \
  -C 100 \                    # 100MB씩 파일 분할
  -W 5 \                      # 최대 5개 파일 순환 (링 버퍼)
  -w /tmp/capture.pcap \
  "host 10.0.0.5 and port 8080"

# 2. 로컬로 복사
scp user@server:/tmp/capture.pcap ~/Desktop/

# 3. Wireshark로 열기
open ~/Desktop/capture.pcap   # macOS

# 실시간 파이프 (SSH + Wireshark 실시간 분석)
ssh user@server "tcpdump -i eth0 -nn -s0 -w - 'port 8080'" | \
  wireshark -k -i -
```

---

## 4. Wireshark 활용

### 설치

```bash
# macOS
brew install --cask wireshark

# Ubuntu
sudo apt install wireshark
sudo usermod -aG wireshark $USER
```

### 핵심 필터 (Display Filter)

```
# TCP 분석
tcp.flags.syn == 1              # SYN 패킷
tcp.flags.rst == 1              # RST 패킷 (비정상 종료)
tcp.analysis.retransmission     # 재전송 패킷
tcp.analysis.duplicate_ack      # 중복 ACK (패킷 손실 신호)
tcp.analysis.zero_window        # 수신 버퍼 꽉 참 (흐름 제어)
tcp.analysis.fast_retransmission

# HTTP
http.request.method == "GET"
http.response.code >= 500       # 서버 에러
http.request.uri contains "/api"

# DNS
dns.flags.rcode == 3            # NXDOMAIN
dns.qry.name contains "example.com"

# TLS
tls.handshake.type == 1         # Client Hello
tls.handshake.type == 2         # Server Hello
tls.alert                       # TLS 경고

# MQTT (자동 디코딩)
mqtt.msgtype == 3               # PUBLISH
mqtt.topic contains "sensor"

# 특정 IP 간 통신
ip.addr == 192.168.1.100 && ip.addr == 10.0.0.5

# 포트
tcp.port == 8080
```

### Statistics 메뉴 활용

```
Statistics → Conversations
  → TCP 연결별 패킷 수, 바이트, 지속 시간 한눈에 보기
  → 비정상적으로 많은 재연결 = 연결 불안정 신호

Statistics → IO Graph
  → 시간대별 트래픽 그래프
  → 스파이크 = DDoS 또는 대량 재전송

Statistics → TCP Stream Graph → Time-Sequence Graph (Stevens)
  → 특정 연결의 TCP 시퀀스 번호 그래프
  → 손실·재전송 패턴 시각화

Analyze → Expert Information
  → 경고(노란색), 에러(빨간색) 자동 분류
  → 재전송, 중복 ACK, RST 등 한눈에 파악
```

### Follow TCP/HTTP Stream

```
특정 패킷 우클릭 → Follow → TCP Stream
  → 해당 연결의 요청/응답 전체를 텍스트로 재조합
  → HTTP 헤더, 바디 내용 직접 확인

Follow → HTTP Stream
  → HTTP 레이어만 추출 (gzip 압축 자동 해제)
```

---

## 5. 트러블슈팅 시나리오 → 도구 선택 가이드

| 증상 | 원인 가설 | 도구 | 필터 |
|------|----------|------|------|
| Connection refused | 포트 닫힘/방화벽 DROP | tcpdump | `tcp[tcpflags] & tcp-rst != 0` |
| 간헐적 타임아웃 | 패킷 손실·재전송 | tcpdump → Wireshark | `tcp.analysis.retransmission` |
| TLS 핸드셰이크 실패 | 인증서·SNI·버전 불일치 | Wireshark | `tls.handshake` |
| DNS 해석 안 됨 | DNS 서버·도메인 문제 | tcpdump | `udp port 53` |
| 로드밸런서 헬스체크 실패 | 응답 지연·포트 불일치 | tcpdump | `port 8080 and tcp-syn` |
| MQTT 메시지 안 옴 | QoS·토픽·연결 끊김 | Wireshark | `mqtt` 자동 디코딩 |
| HTTP 응답 느림 | 서버 처리 지연·네트워크 지연 | Wireshark | Time-Sequence Graph |
| 캐시 서버 미스 | Redis 연결 실패 | tcpdump | `host redis-ip and port 6379` |

---

## 6. 권한 및 운영 주의사항

```bash
# root 없이 실행 (필요한 능력만 부여)
sudo setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)
tcpdump -i eth0 ...   # sudo 없이 가능

# 컨테이너 내에서 tcpdump (특권 필요)
docker exec -it --privileged container_name tcpdump -i eth0

# K8s Pod에서 디버깅
kubectl debug -it pod/myapp --image=nicolaka/netshoot -- tcpdump -i eth0

# netshoot — 네트워크 디버깅 전용 컨테이너
kubectl run netshoot --rm -it \
  --image=nicolaka/netshoot \
  --overrides='{"spec":{"hostNetwork":true}}' \
  -- tcpdump -i eth0 "port 8080"

# 개인정보 주의: 암호화 안 된 HTTP 패킷에 사용자 데이터 포함 가능
# 캡처 파일 보관 시 접근 제한 필수
```

---

## 7. 관련
- [[Kernel-Parameters]] — net.core.rmem_max 등 네트워크 커널 파라미터
- [[../network/BGP-Anycast]] — 네트워크 레이어 이슈
- [[../observability/PLG-Stack]] — 로그 기반 트러블슈팅 (패킷 캡처 이전 단계)
