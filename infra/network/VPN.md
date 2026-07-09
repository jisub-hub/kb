---
tags:
  - network
  - vpn
  - ipsec
  - ssl-vpn
  - security
created: 2026-06-15
---

# VPN — IPSec · SSL VPN · 터널링

> [!summary] 한 줄 요약
> **VPN(Virtual Private Network)**: 공용 인터넷 위에 **암호화된 전용 터널**을 만들어 사설망처럼 통신하는 기술. 온프레미스-클라우드 연결(IPSec Site-to-Site), 개발자 원격 접속(SSL VPN) 등에 사용된다.

---

## 1. VPN 개념

```
[본사 네트워크] ←── 인터넷 (암호화 터널) ───→ [클라우드 VCN/VPC]
                        IPSec / SSL VPN
```

- 공용 인터넷을 경유하지만 **암호화로 도청·변조 방지**
- 양 끝단(엔드포인트)만 키를 알고 있어 중간자 공격 차단
- 물리적 전용선(FastConnect/Direct Connect) 대비 저비용

---

## 2. IPSec (IP Security)

네트워크 계층(L3)에서 패킷을 암호화·인증하는 프로토콜 모음.

### 동작 모드

| 모드 | 암호화 범위 | 용도 |
|---|---|---|
| **Transport 모드** | IP 페이로드만 암호화 (헤더 유지) | 호스트 간 직접 통신 |
| **Tunnel 모드** | 원본 IP 패킷 전체를 새 IP 패킷으로 캡슐화 | **Site-to-Site VPN** (일반적) |

### IPSec 구성 요소

| 프로토콜 | 역할 |
|---|---|
| **IKE** (Internet Key Exchange) | 키 협상 및 SA(Security Association) 수립. IKEv2가 현행 표준 |
| **AH** (Authentication Header) | 무결성 보장 (암호화 없음, 거의 안 씀) |
| **ESP** (Encapsulating Security Payload) | **암호화 + 무결성** (실질적으로 ESP만 사용) |

### Site-to-Site IPSec VPN (온프레미스 ↔ 클라우드)

```
[온프레미스 라우터/방화벽]
        ↕ IKE 협상 (UDP 500/4500)
[클라우드 VPN Gateway (DRG/VGW/VPN GW)]
        ↕ ESP 터널 (암호화 패킷)
```

**클라우드 설정 예시 (OCI):**
```
1. DRG(Dynamic Routing Gateway) 생성 → VCN에 연결
2. IPSec Connection 생성
   - 온프레미스 공인 IP 지정
   - 공유 비밀키(Pre-Shared Key) 또는 인증서 설정
3. Tunnel 2개 생성 (HA 목적 — 터널 1개 장애 시 자동 전환)
4. 온프레미스 장비에 동일 설정 입력
5. 라우팅 테이블 업데이트 (온프레미스 대역 → DRG)
```

**IKE Phase 협상:**
```
Phase 1 (ISAKMP SA): 암호화 알고리즘, 인증 방식 협상 → 안전한 채널 수립
Phase 2 (IPSec SA): 실제 데이터 암호화 터널 파라미터 협상
```

---

## 3. SSL VPN (Client-to-Site VPN)

개발자·운영자가 **개인 PC에서 클라우드/사내망에 원격 접속**할 때 사용.

```
[개발자 노트북] ──HTTPS(443)──→ [SSL VPN 서버] ──→ [사내 네트워크/클라우드]
```

- **TLS(SSL) 위에서 동작** → 방화벽 친화적 (443 포트 사용, 대부분 방화벽 통과)
- 브라우저 또는 VPN 클라이언트 앱 사용
- IPSec Site-to-Site와 달리 **사용자별 인증** (LDAP, MFA 등)

| | IPSec Site-to-Site | SSL VPN (Client VPN) |
|---|---|---|
| 대상 | 사이트 간 항상 연결 | 개별 사용자 원격 접속 |
| 인증 | Pre-Shared Key 또는 인증서 | 사용자 ID/PW + MFA |
| 포트 | UDP 500, 4500 | TCP/UDP 443 |
| 설정 | 라우터/방화벽 설정 | 클라이언트 앱 설치 |
| 예시 | 온프레미스 ↔ AWS VPC | 재택 개발자 → 내부망 |

**클라우드 SSL VPN 서비스:**
- OCI: OpenVPN 기반 자체 구성
- AWS: Client VPN (OpenVPN 프로토콜)
- Azure: Point-to-Site VPN Gateway

---

## 4. WireGuard

IPSec/OpenVPN 대비 **단순하고 고성능**인 최신 VPN 프로토콜.

```bash
# WireGuard 서버 설정 예시 (Linux)
[Interface]
Address = 10.100.0.1/24
ListenPort = 51820
PrivateKey = <서버 개인키>

[Peer]
PublicKey = <클라이언트 공개키>
AllowedIPs = 10.100.0.2/32    # 이 클라이언트가 받을 IP
```

| | IPSec | OpenVPN | WireGuard |
|---|---|---|---|
| 표준 | IETF | 오픈소스 | 리눅스 커널 내장 (5.6+) |
| 성능 | 높음 | 중간 | **최고** |
| 설정 복잡도 | 높음 | 중간 | **낮음** |
| 포트 | UDP 500/4500 | TCP/UDP 1194/443 | UDP 51820 (기본) |
| 암호화 | AES-256, SHA-256 | AES-256, SHA-256 | ChaCha20, Poly1305 |
| 모바일 | 지원 | 지원 | **우수** |

---

## 5. VPN 선택 기준

| 상황 | 권장 |
|---|---|
| 온프레미스 ↔ 클라우드 영구 연결 | IPSec Site-to-Site VPN (또는 전용선 [[Gateway]]) |
| 개발자 원격 접속 | SSL VPN (AWS Client VPN, OpenVPN) |
| 소규모/개인 인프라 | WireGuard |
| 고대역폭·저지연 필요 | 전용선 (FastConnect/Direct Connect) |
| 임시 연결·테스트 | SSH 터널 (`ssh -L`) |

---

## 6. 관련
- [[Gateway]] · [[Security-ACG-NSG]] · [[VCN-VPC]]
