---
tags:
  - network
  - bandwidth
  - throughput
  - latency
created: 2026-06-15
---

# 네트워크 대역폭 & 성능 지표

> [!summary] 한 줄 요약
> **대역폭**: 단위 시간당 전송 가능한 최대 데이터 양(Mbps/Gbps). **처리량**: 실제 전송된 양. **지연**: 패킷 이동 시간. 인프라 설계 시 세 가지를 함께 고려해야 한다.

---

## 1. 핵심 지표 3가지

| 지표 | 정의 | 단위 | 영향 요소 |
|---|---|---|---|
| **대역폭 (Bandwidth)** | 최대 전송 용량 | Mbps, Gbps | 네트워크 장비, 회선 등급 |
| **처리량 (Throughput)** | 실제 전송된 데이터 양 | Mbps | 혼잡도, 패킷 손실, 오버헤드 |
| **지연 (Latency)** | 패킷이 A→B 이동 시간 | ms | 거리(빛의 속도), 홉 수, 처리 시간 |
| **지터 (Jitter)** | 지연의 변동폭 | ms | 네트워크 혼잡, QoS 설정 |
| **패킷 손실 (Packet Loss)** | 전송 중 유실된 패킷 비율 | % | 혼잡, 오류, 장비 과부하 |

> [!tip] 대역폭 ≠ 처리량
> 1Gbps 회선이라도 TCP 오버헤드, TLS 암호화, 패킷 손실 재전송으로 실제 처리량은 60~80%가 일반적.

---

## 2. 대역폭 단위

```
bit (b) → Kilobit (Kb) → Megabit (Mb) → Gigabit (Gb) → Terabit (Tb)
                1Gbps = 1,000Mbps = 1,000,000Kbps

파일 크기와 혼동 주의:
  대역폭: 소문자 b = bit (bps)
  파일 크기: 대문자 B = Byte (1 Byte = 8 bit)

1Gbps 회선으로 1GB 파일 전송 이론 시간:
  1GB = 8Gb → 8Gb ÷ 1Gbps = 8초
```

---

## 3. 클라우드 네트워크 대역폭

### VM 네트워크 성능

클라우드 VM은 인스턴스 타입에 따라 네트워크 대역폭이 다르다.

| 클라우드 | 인스턴스 | 대역폭 |
|---|---|---|
| OCI | VM.Standard.E4.Flex (1 OCPU) | 1 Gbps |
| OCI | VM.Standard.E4.Flex (4 OCPU) | 4 Gbps |
| AWS | t3.medium | 최대 5 Gbps (burstable) |
| AWS | c5n.xlarge | 최대 25 Gbps |
| AWS | c5n.18xlarge | 100 Gbps |

### 클라우드 간 트래픽 비용

| 트래픽 방향 | 비용 |
|---|---|
| 인터넷 → 클라우드 (인바운드) | 무료 (대부분) |
| 클라우드 → 인터넷 (아웃바운드) | **유료** (GB당 과금) |
| 같은 AZ 내 VM 간 | 무료 또는 저렴 |
| 다른 AZ 간 | 유료 (Data Transfer) |
| 다른 리전 간 | 유료 (더 비쌈) |

> [!tip] 아키텍처에서 비용 줄이기
> - 같은 AZ에 통신이 잦은 서비스 배치
> - Service Gateway로 S3/Object Storage 접근 시 인터넷 트래픽 비용 절감
> - CDN(CloudFront, OCI Edge) 활용으로 아웃바운드 트래픽 분산

---

## 4. 지연(Latency) 기준값

```
서울 데이터센터 내 (같은 IDC): < 1ms
서울 ↔ 부산 (국내 리전 간): 5~15ms
서울 ↔ 도쿄 (해외 리전): 30~50ms
서울 ↔ 미국 서부: 150~200ms
서울 ↔ 유럽: 250~300ms
```

**빛의 속도 한계:**
- 광섬유에서 빛 속도 ≈ 200,000 km/s
- 서울↔도쿄 ≈ 1,200km → 이론 최솟값 ≈ 6ms (실제는 홉/처리 시간 추가)

---

## 5. 애플리케이션 관점

### DB 쿼리 지연 영향

```
같은 서버 내 메모리 접근: < 0.1ms
같은 AZ DB 쿼리: 0.5~2ms
다른 AZ DB 쿼리: 2~5ms
원격 리전 DB: 50~200ms (권장하지 않음)
```

→ DB는 **WAS와 같은 AZ에 배치** 권장.

### HTTP API 성능 목표

| P50 | P95 | P99 | 상태 |
|---|---|---|---|
| < 50ms | < 200ms | < 500ms | 양호 |
| < 100ms | < 500ms | < 1s | 허용 |
| > 100ms | > 1s | > 2s | 개선 필요 |

---

## 6. 대역폭 테스트 도구

```bash
# iperf3 — 두 서버 간 처리량 측정
# 서버 측
iperf3 -s

# 클라이언트 측
iperf3 -c <서버IP> -t 30        # 30초 TCP 테스트
iperf3 -c <서버IP> -u -b 100M   # 100Mbps UDP 테스트

# ping — 지연 측정
ping -c 100 <서버IP>
# min/avg/max/mdev 확인

# traceroute — 경로 홉별 지연 확인
traceroute <서버IP>

# curl — HTTP 응답 시간
curl -w "@curl-format.txt" -o /dev/null -s https://api.example.com
```

---

## 7. CDN (Content Delivery Network)

정적 콘텐츠를 전 세계 엣지 서버에 캐싱 → 최종 사용자 지연 최소화.

| 클라우드 | CDN 서비스 |
|---|---|
| OCI | Oracle CDN (Akamai 기반) |
| AWS | CloudFront |
| Azure | Azure CDN |
| 독립형 | Cloudflare, Akamai, Fastly |

```
사용자 (서울) → Cloudflare 엣지 (서울) → 캐시 히트 → < 5ms 응답
                                        → 캐시 미스 → Origin (AWS us-east-1) → 180ms
```

---

## 8. 관련
- [[Network-Basics]] · [[Gateway]] · [[VPN]] · [[HA-Architecture]]
