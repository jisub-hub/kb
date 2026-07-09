---
tags:
  - network
  - api-gateway
  - l7
  - security
created: 2026-06-15
---

# API Gateway

> [!summary] 한 줄 요약
> 외부 클라이언트와 백엔드 사이의 **API 관리 전담 게이트웨이**. 인증·Rate Limit·라우팅·로깅을 중앙에서 처리해 백엔드에서 공통 로직을 제거한다. Load Balancer와 역할이 다르다.

---

## 1. API Gateway vs Load Balancer

| | Load Balancer | API Gateway |
|---|---|---|
| 동작 계층 | L4 (NLB) / L7 (ALB) | L7 (HTTP 전용) |
| 주 역할 | 트래픽 **분산** | API **관리·정책** |
| 인증/인가 | ❌ 없음 | ✅ JWT, OAuth2, API Key |
| Rate Limiting | ❌ 없음 | ✅ 클라이언트별 제한 |
| 라우팅 | IP/포트 기반 | URL/메서드 기반 |
| 로깅 | 접근 로그 (IP, 바이트) | API 호출 상세 (호출자, 응답코드, 지연) |
| 주 대상 | 내부 서버 분산 | **외부 API 노출** |
| 서킷 브레이커 | ❌ | ✅ (일부 제품) |

---

## 2. 아키텍처 위치

```
인터넷
  └→ API Gateway (인증·Rate Limit·라우팅)
        └→ Load Balancer (트래픽 분산)
              ├→ WEB 서버 1
              └→ WEB 서버 2
                    └→ WAS
                          └→ DB
```

- API GW는 **외부 트래픽의 첫 번째 관문**
- LB는 API GW 뒤에서 백엔드 서버들 간 분산 담당

---

## 3. 주요 기능

### 인증 / 인가
```
클라이언트 → API GW (JWT 검증) → 통과 시 백엔드 전달
                               → 실패 시 401/403 즉시 반환
```
- JWT, OAuth2 Access Token, API Key 검증을 GW에서 일괄 처리
- 백엔드 서버에서 인증 로직을 각자 구현할 필요 없음

### Rate Limiting
```
특정 클라이언트가 1분에 100회 초과 요청 → 429 Too Many Requests 반환
```
- DDoS 방어, API 남용 방지
- IP별, API Key별, 사용자별 등 다양한 기준 설정

### 라우팅 / API 버전 관리
```
/v1/users → WAS-v1 cluster
/v2/users → WAS-v2 cluster
/api/orders → Order Service
/api/members → Member Service
```
- URL Path 기반으로 MSA 각 서비스로 분기
- 버전 전환을 GW에서 처리 → 백엔드 무중단 버전 업

### 로깅 / 모니터링
- API 호출자 IP, 사용자 ID, 경로, 응답코드, 지연시간 기록
- 이상 트래픽 감지, 사용량 분석, 장애 추적에 활용

### 요청/응답 변환
- 헤더 추가/제거 (예: 인증 후 `X-User-Id` 헤더 백엔드에 주입)
- 요청 형식 변환 (XML → JSON 등) — 레거시 시스템 연동 시 유용

---

## 4. 클라우드 제품 & 오픈소스

| 구분 | 제품 |
|---|---|
| OCI | OCI API Gateway |
| AWS | Amazon API Gateway (REST/HTTP/WebSocket) |
| Azure | Azure API Management |
| 오픈소스 | Kong, NGINX (Plus/OSS), Envoy, Traefik |

**Kong**: Lua 플러그인 기반, 커뮤니티 버전 무료. 인증·Rate Limit·로깅 플러그인 풍부.
**NGINX**: 직접 설정 파일로 라우팅·Rate Limit 구현. 소규모에 적합.

---

## 5. 주의 사항

> [!warning] API Gateway는 SPOF가 될 수 있다
> 모든 트래픽이 GW를 거치므로 GW 자체가 장애나면 전체 서비스 중단. 클라우드 관리형 서비스 사용 또는 GW 자체 HA 구성 필수.

> [!tip] 내부 서비스 간 통신에는 API GW 대신 Service Mesh
> API GW는 **외부 클라이언트 → 백엔드** 용도. MSA 내부 서비스 간 통신은 **Istio 같은 Service Mesh**가 더 적합.

---

## 6. 관련
- [[Load-Balancer]] · [[Security-ACG-NSG]] · [[VCN-VPC]]
