---
tags:
  - security
  - network
  - ddos
  - attack
  - defense
created: 2026-06-16
---

# 네트워크 공격 기법 및 방어

> [!summary] 한 줄 요약
> **DDoS**는 다수의 소스로 대역폭·CPU를 소진시켜 서비스를 마비시키는 공격. UDP Flood/Ping of Death(볼륨 계층), SYN Flood(프로토콜 계층), HTTP Flood(애플리케이션 계층)로 분류. Reflect/Amplification이 현대 최대 규모 공격 벡터.

---

## 1. DDoS 공격 분류 체계

```
OSI 계층별 분류:

L7 (Application)  HTTP Flood, Slowloris, RUDY, DNS Query Flood
                  → CPU·메모리 소진
L4 (Transport)    SYN Flood, UDP Flood, ACK Flood
                  → 연결 테이블 소진
L3 (Network)      ICMP Flood, Ping of Death, Smurf, IP Fragmentation
                  → 대역폭 소진, 버퍼 오버플로
L2 (Data Link)    MAC Flood, ARP Spoofing
                  → 스위치 CAM 테이블 고갈 (로컬망)
```

```
DoS vs DDoS:
  DoS  (Denial of Service):       단일 소스
  DDoS (Distributed DoS):         수천~수백만 소스 (봇넷)
  DRDoS (Distributed Reflection): 증폭 서버를 중간에 이용 (소스 위장)
```

---

## 2. UDP Flood

### 원리

```
UDP = 비연결형 프로토콜 → 핸드셰이크 없음 → 위조된 소스 IP 사용 가능

[공격 흐름]
  봇넷 → 임의 포트로 대용량 UDP 패킷 전송 → 피해 서버
          (소스 IP 스푸핑 가능)

피해 서버 처리 과정:
  1. UDP 패킷 수신
  2. 해당 포트에 서비스 없음 → ICMP Port Unreachable 응답
  3. 무수한 패킷 → ICMP 생성 반복 → CPU/대역폭 포화
```

```
특징:
  · 소스 IP 위조 가능 → 공격자 추적 어려움
  · 패킷 크기 최대화 (65,507바이트) → 대역폭 소진 극대화
  · 증폭에 활용 가능 (DNS, NTP, SSDP UDP 기반)

실측:
  Gbps~Tbps 규모 공격 가능 (봇넷 규모에 따라)
  Mirai 봇넷(2016): 피크 620 Gbps, DNS 서버 무력화
```

### 방어

```
[네트워크 계층]
  · 업스트림 ISP에서 UDP Rate Limiting
  · uRPF (Unicast Reverse Path Forwarding) — 소스 IP 위조 차단
  · ACL로 비정상 UDP 포트 차단 (사용 안 하는 포트)

[서버 계층]
  · iptables UDP 속도 제한
    iptables -A INPUT -p udp --dport 53 -m limit --limit 100/s -j ACCEPT
    iptables -A INPUT -p udp --dport 53 -j DROP

  · 필요한 UDP 포트만 열기 (게임 서버: 특정 포트, DNS: 53만)

[서비스 계층]
  · Anycast + 스크러빙 센터 (Cloudflare, Akamai)
  · BGP Blackholing — 공격 트래픽 전체를 null route로 폐기
```

---

## 3. Ping of Death

### 원리

```
IPv4 패킷 최대 크기: 65,535 바이트
ICMP Echo (ping) 최대 데이터: 65,507 바이트

[취약점]
  IP 패킷은 1,500바이트(MTU) 단위로 분할(Fragmentation)
  수신 측에서 재조합(Reassembly)

  공격: 65,535바이트 초과 패킷을 분할하여 전송
        재조합 시 버퍼 오버플로 → 커널 크래시 / BSOD

ping -l 65510 victim.com  (Windows 취약 버전)
```

```
역사:
  · 1990년대 중반 Windows 95/NT, macOS, Linux 커널에 영향
  · 현대 OS: 재조합 버퍼 크기 검사로 패치됨
  · 현재는 과거의 공격이나 임베디드·IoT 기기에 여전히 잔존 가능

변형: ICMP Flood (현재형)
  65,535바이트 이하이지만 대량의 ICMP Echo 패킷 전송
  → 대역폭 소진 (내용보다 양으로 공격)
```

### 방어

```
[현대 OS]: 자동 방어 (커널 패치됨)
[임베디드/IoT 기기]:
  · 펌웨어 업데이트
  · 방화벽에서 외부 ICMP 차단 또는 Rate Limit
  · iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
  · iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
[네트워크]:
  · 경계 라우터에서 Jumbo Frame 외 분할 패킷 검사
  · IPS 시그니처 탐지
```

---

## 4. SYN Flood

### 원리

```
TCP 3-Way Handshake:
  Client → SYN →      Server
  Client ← SYN-ACK ← Server  ← 서버: 세션 테이블에 HALF-OPEN 기록
  Client → ACK →      Server  ← 완료

SYN Flood:
  공격자: SYN 전송 후 ACK 미전송 (또는 소스 IP 위조로 ACK 불가)
  서버: SYN-ACK 전송 후 ACK 대기 → HALF-OPEN 상태 유지
  서버 세션 테이블 가득 참 (기본 backlog ~128~512)
  → 새 연결 수락 불가 → 정상 사용자 연결 거부
```

```bash
# 공격 도구 예시 (교육 목적)
hping3 -S --flood -V -p 80 target.com   # SYN 패킷 플러드
# -S: SYN flag, --flood: 최대속도, -p: 포트
```

### 방어

```
[SYN Cookie] — 근본적 방어
  서버: SYN-ACK에 암호화된 쿠키 포함 (세션 테이블에 저장 안 함)
  클라이언트: ACK에 쿠키 포함
  서버: 쿠키 검증 후 세션 생성
  → Half-open 상태를 서버 메모리에 보관하지 않음

Linux 활성화:
  sysctl -w net.ipv4.tcp_syncookies=1

[기타]
  net.ipv4.tcp_max_syn_backlog=4096   # Backlog 크기 증가
  net.ipv4.tcp_synack_retries=2       # 재전송 횟수 감소
  방화벽 Rate Limit (IP당 SYN 초당 n개)
  Anycast + 스크러빙 센터
```

---

## 5. Reflect & Amplification DDoS

### 원리

```
[Reflection] — 소스 IP 위조로 응답을 피해자에게 반사

  공격자: src_ip=피해자IP, dst=반사서버  → 요청 전송
  반사서버: 피해자 IP로 응답 전송
  → 공격자 IP 노출 없음

[Amplification] — 작은 요청 → 큰 응답으로 증폭

  공격자 → 62바이트 DNS 요청 → DNS 서버
  피해자 ← 3,000바이트 DNS 응답 ← DNS 서버
  증폭 계수: 48배 (DNS ANY 쿼리)
```

### 주요 증폭 프로토콜

| 프로토콜 | 포트 | 증폭 계수 | 설명 |
|---|---|---|---|
| **DNS** | UDP/53 | 28~54배 | ANY 쿼리 또는 대형 레코드 |
| **NTP** | UDP/123 | **556배** | monlist 명령 (서버 접속 목록) |
| **SSDP** | UDP/1900 | 30배 | UPnP 장치 검색 응답 |
| **Memcached** | UDP/11211 | **51,000배** | 2018년 GitHub 1.3Tbps 공격 |
| **CLDAP** | UDP/389 | 56~70배 | MS Active Directory |
| **CharGen** | UDP/19 | 가변 | 무작위 문자 반환 서비스 |

```
역대 최대 DDoS:
  2018년 GitHub: Memcached 증폭, 1.3 Tbps
  2020년 AWS: 2.3 Tbps
  2022년 Cloudflare: 26M RPS (HTTP/3 기반)
  2023년 HTTP/2 Rapid Reset: 201M RPS
```

### 방어

```
[반사 서버 측 (서비스 제공자)]
  DNS:       응답 속도 제한, 오픈 리졸버 차단 (자사 IP만 허용)
  NTP:       monlist 비활성화 (ntpdc -c monlist 제한)
  Memcached: UDP 포트 비활성화, 방화벽 차단
  SSDP:      외부 인터페이스에서 UDP/1900 차단

[피해자 측]
  · Anycast 라우팅 + 스크러빙 센터
  · BGP Blackholing
  · Cloudflare/Akamai/AWS Shield 등 CDN+DDoS 완화 서비스
  · 대역폭 초과 자동 업스트림 클린징

[uRPF (Unicast Reverse Path Forwarding)]
  ISP 레벨에서 소스 IP 위조 차단 (BCP38)
  → Reflection 공격의 근본 원인 제거
  → 아직 전 세계적 적용률 낮음 (~30%)
```

---

## 6. HTTP Flood (Layer 7 DDoS)

### 원리

```
정상 HTTP 요청처럼 보이는 대량 요청:
  · 단순 GET/POST 플러드
  · 무거운 엔드포인트 타겟 (검색, 로그인, 대용량 파일 생성)
  · Cache Bypass: 매 요청에 랜덤 파라미터 추가 → CDN 캐시 무력화
    GET /search?q=keyword&_=1234567890 (매번 다른 타임스탬프)

특징:
  · 소스가 분산되어 있고 정상 트래픽과 구별 어려움
  · IP 차단만으로는 부족 → 봇 탐지 필요
  · HTTPS라도 막을 수 없음 (암호화된 L7 공격)
```

### 방어

```
[WAF (Web Application Firewall)]
  · 요청 속도 제한 (IP당 분당 n회)
  · User-Agent 검증 (알려진 봇 시그니처)
  · Challenge-Response (JS 검사, CAPTCHA)

[Rate Limiting (Nginx/Spring)]
  # Nginx
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
  limit_req zone=api burst=20 nodelay;

  # Spring Boot
  @Bean
  public RedisRateLimiter redisRateLimiter() {
      return new RedisRateLimiter(10, 20);  // replenishRate, burstCapacity
  }

[Bot Detection]
  · Cloudflare Bot Management
  · Device fingerprinting
  · Behavioral analysis (마우스 움직임, JS 실행 여부)

[Slowloris / RUDY 방어]
  → Nginx: client_header_timeout 10s; client_body_timeout 10s;
  → 연결 완료까지 최대 대기 시간 제한
```

---

## 7. Slowloris

### 원리

```
HTTP 헤더를 천천히 전송해 서버 연결을 점유:

  공격자: GET / HTTP/1.1\r\n
          Host: victim.com\r\n
          X-Custom: aaa\r\n
          (헤더 끝을 의미하는 \r\n\r\n 전송 안 함)
          ... 30초마다 헤더 한 줄씩 추가 ...

서버: 헤더가 완성될 때까지 연결 유지 (HTTP 스펙)
     → 수천 개 연결 점유 → 새 연결 수락 불가

특징:
  · 대역폭 거의 사용 안 함 (저속 공격)
  · 단일 저사양 PC로 Apache 서버 마비 가능
  · Nginx: 이벤트 드리븐 → 상대적으로 안전
  · Apache(스레드 기반): 매우 취약
```

### 방어

```
Apache:
  LoadModule reqtimeout_module modules/mod_reqtimeout.so
  RequestReadTimeout header=10-20,minrate=500 body=20

Nginx (기본적으로 방어됨):
  client_header_timeout  10s;
  keepalive_timeout      30s;
  client_max_body_size   10m;

일반:
  연결 타임아웃 설정
  IP당 동시 연결 수 제한
  HAProxy / Nginx를 Apache 앞단에 배치
```

---

## 8. 방어 아키텍처 (계층별)

```
[인터넷]
     │
     ▼
[ISP / 업스트림]
  · uRPF (소스 IP 위조 차단)
  · BGP Blackholing (대규모 공격 시 특정 IP null route)
     │
     ▼
[CDN / DDoS 스크러빙 센터]
  Cloudflare / Akamai / AWS Shield
  · Anycast — 트래픽을 가장 가까운 PoP으로 분산
  · 스크러빙 — 악성 트래픽 필터링 후 정상 트래픽만 전달
  · L3/L4/L7 공격 모두 흡수 가능
     │
     ▼
[Load Balancer / WAF]
  · L7 Rate Limiting
  · IP Reputation 차단
  · Bot Challenge (JS 검사, CAPTCHA)
     │
     ▼
[서버]
  · SYN Cookie 활성화
  · OS 레벨 Rate Limit (iptables)
  · 애플리케이션 레벨 Circuit Breaker
```

### 비용 대비 효과

| 방어 수단 | 비용 | 방어 계층 | 적용 난이도 |
|---|---|---|---|
| **Cloudflare Free/Pro** | 무료~$20/월 | L3~L7 | ⭐ (DNS만 변경) |
| **AWS Shield Standard** | 무료 | L3/L4 | ⭐ (자동) |
| **AWS Shield Advanced** | $3,000/월 | L3~L7 | ⭐⭐ |
| **SYN Cookie** | 무료 | L4 | ⭐ (sysctl) |
| **Nginx Rate Limit** | 무료 | L7 | ⭐⭐ |
| **Anycast 자체 구축** | 높음 | L3/L4 | ⭐⭐⭐⭐⭐ |

---

## 9. 관련
- [[Cryptography]] · [[Web-Attacks]] · [[../../infra/architecture/Public-Network-Security]]
