---
tags:
  - network
  - load-balancer
  - l4
  - l7
  - ha
created: 2026-06-15
---

# Load Balancer

> [!summary] 한 줄 요약
> **여러 백엔드 서버에 트래픽을 분산**해 단일 진입점을 제공하고, 장애 서버를 자동 제외한다. 동작 계층에 따라 L4 NLB와 L7 ALB로 나뉜다.

---

## 1. 핵심 역할

- **트래픽 분산**: 요청을 여러 VM에 나눠 처리 → 스케일 아웃 가능
- **단일 진입점(Single Endpoint)**: 클라이언트는 LB IP 하나만 알면 됨
- **Health Check → 자동 제외**: 응답 없는 VM은 자동으로 풀에서 제거, 복구 시 복귀
- **가용성 확보**: VM 한 대 장애 시 나머지로 자동 전환 (무중단)

---

## 2. L4 NLB (Network Load Balancer)

```
클라이언트 → NLB (TCP/UDP 레벨) → Backend VM
```

- **TCP/UDP** 레벨 분산. IP + Port 기준 라우팅
- HTTP 헤더·URL 내용은 볼 수 없음
- **초저지연, 극고성능** (수백만 req/s 처리 가능)
- 주 용도: 게임 서버, 실시간 스트리밍, 비HTTP 프로토콜

| 클라우드 | L4 LB 서비스 |
|---|---|
| OCI | Network Load Balancer (NLB) |
| AWS | Network Load Balancer (NLB) |
| Azure | Azure Load Balancer |

---

## 3. L7 ALB (Application Load Balancer)

```
클라이언트 → ALB (HTTP/HTTPS 레벨) → Backend VM
           ↳ URL Path / Host / Header 분석 가능
```

- **HTTP/HTTPS** 레벨 분산
- URL Path, Host Header, Query String 기준 **콘텐츠 기반 라우팅**
- **SSL Termination**: HTTPS 암복호화를 LB에서 처리 → 백엔드는 HTTP로 통신
- WAF(Web Application Firewall) 연동 가능
- 주 용도: 웹 애플리케이션, REST API, MSA 서비스별 라우팅

| 클라우드 | L7 LB 서비스 |
|---|---|
| OCI | Load Balancer (Flexible) |
| AWS | Application Load Balancer (ALB) |
| Azure | Application Gateway |

---

## 4. 분산 알고리즘

| 알고리즘 | 동작 방식 | 적합한 상황 |
|---|---|---|
| **Round Robin** | 순서대로 균등 배분 | 요청 처리 시간이 유사할 때 (기본값) |
| **Least Connections** | 현재 연결 수가 적은 서버 우선 | 처리 시간 편차가 클 때 |
| **IP Hash** | 클라이언트 IP로 동일 서버 고정 | 세션 유지가 필요할 때 (Sticky Session 대안) |
| **Weighted Round Robin** | 서버별 가중치 다르게 설정 | 서버 스펙이 다를 때 |

---

## 5. Health Check

LB가 주기적으로 백엔드 서버에 요청을 보내 정상 여부 확인.

```
LB → GET /health → HTTP 200 OK  ✅ 정상 → 풀 유지
LB → GET /health → 타임아웃     ❌ 비정상 → 풀에서 제외
```

**설정 항목:**
| 항목 | 설명 | 일반적 권장값 |
|---|---|---|
| 경로 | 체크할 URL | `/health`, `/actuator/health` |
| 프로토콜 | HTTP / TCP | HTTP (응답 코드 확인 가능) |
| 간격 | 체크 주기 | 10~30초 |
| 타임아웃 | 응답 대기 시간 | 5~10초 |
| 임계값 (unhealthy) | 연속 실패 횟수 | 2~3회 |
| 임계값 (healthy) | 연속 성공 횟수 | 2~3회 |

> [!tip] Spring Boot Actuator 연동
> `management.endpoints.web.exposure.include=health` 설정 후 `/actuator/health` 를 LB Health Check 경로로 사용하면 DB 커넥션·디스크까지 종합 상태 반환.

---

## 6. SSL Termination

```
클라이언트 ──HTTPS──→ ALB (인증서 보관, 복호화) ──HTTP──→ 백엔드
```

- HTTPS 암복호화(TLS) 처리를 ALB가 담당 → **백엔드 서버 CPU 부하 감소**
- 인증서를 LB 1곳에서 관리 → 갱신 편리
- 백엔드는 HTTP로 통신하므로, **백엔드-LB 구간도 암호화**하려면 End-to-End SSL 설정 필요

---

## 7. L4 vs L7 선택 기준

| | L4 NLB | L7 ALB |
|---|---|---|
| 프로토콜 | TCP/UDP 모두 | HTTP/HTTPS만 |
| 라우팅 | IP + 포트만 | URL, 헤더, 경로별 |
| SSL Termination | ❌ (Pass-through 방식) | ✅ |
| WAF 연동 | ❌ | ✅ |
| 성능 | 극고성능 | 고성능 |
| 사용 사례 | 게임, IoT, DB LB | 웹앱, API, MSA |

---

## 8. 관련
- [[API-Gateway]] · [[HA-Architecture]] · [[VCN-VPC]]
