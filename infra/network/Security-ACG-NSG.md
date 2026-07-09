---
tags:
  - network
  - security
  - acg
  - nsg
  - firewall
  - defense-in-depth
created: 2026-06-15
---

# 보안 제어 — ACG · NSG · VM 방화벽 · Defense in Depth

> [!summary] 한 줄 요약
> 클라우드 네트워크 보안은 **계층별 다중 방어**. 서브넷 단위 NSG/NACL → VM 단위 ACG/SG → OS 레벨 방화벽 순서로 트래픽을 걸러낸다.

---

## 1. ACG / Security Group — 인스턴스 단위 방화벽

```
인터넷 → (NSG 통과) → VM [ACG 적용됨]
```

- VM(인스턴스)에 직접 붙이는 **가상 방화벽**
- **Stateful**: 인바운드를 허용하면 해당 연결의 아웃바운드 응답은 자동 허용
- 허용 규칙만 작성 (명시적 거부 없음 → 나머지는 모두 암묵적 차단)
- VM 하나에 여러 ACG 동시 적용 가능

### ACG 룰 작성 예시 (웹서버)

| 방향 | 프로토콜 | 포트 | 소스/대상 | 설명 |
|---|---|---|---|---|
| Inbound | TCP | 22 | `10.0.0.0/16` | **내부망에서만** SSH 허용 |
| Inbound | TCP | 80 | `0.0.0.0/0` | 전체 HTTP 허용 |
| Inbound | TCP | 443 | `0.0.0.0/0` | 전체 HTTPS 허용 |
| Outbound | TCP | 443 | `0.0.0.0/0` | 외부 API 호출 |
| Outbound | TCP | 3306 | `10.0.3.0/24` | DB 서브넷으로만 MySQL |

### 3-Tier 보안 정책 설계

| 계층 | ACG 이름 | 인바운드 허용 | 아웃바운드 허용 |
|---|---|---|---|
| Web | web-acg | 80/443 ← `0.0.0.0/0` | 8080 → WAS SN |
| WAS | was-acg | 8080 ← Web SN만 | 3306 → DB SN |
| DB | db-acg | 3306 ← WAS SN만 | (최소화) |

> [!danger] 최소 권한 원칙 (Least Privilege)
> - `0.0.0.0/0` 으로 22(SSH) 포트 오픈 → **절대 금지**
> - DB 포트(3306/5432)를 인터넷에 노출 → **절대 금지**
> - SSH는 반드시 특정 IP(Bastion Host, VPN 대역)로만 제한

---

## 2. NSG / NACL — 서브넷 단위 방화벽

```
인터넷 → IGW → [NSG/NACL 적용] → 서브넷 → VM [ACG 적용]
                   1차 필터                      2차 필터
```

- 서브넷 전체에 적용 → 서브넷에 들어오고 나가는 트래픽을 제어
- **NACL (AWS)**: **Stateless** — 인바운드 허용해도 아웃바운드 응답 별도 허용 필요
- **NSG (OCI/Azure VM 단위)**: 설정 방식은 ACG와 유사하나 서브넷 또는 그룹 단위로 적용

### Stateful vs Stateless 비교

| | ACG / Security Group | NACL (AWS) |
|---|---|---|
| 상태 관리 | Stateful | Stateless |
| 응답 트래픽 | 자동 허용 | 별도 허용 규칙 필요 |
| 허용/거부 | 허용만 | 허용·거부 모두 가능 |
| 적용 단위 | 인스턴스(VM) | 서브넷 |
| 규칙 번호 | 없음 (전체 적용) | 있음 (낮은 번호 우선) |

### NACL (Stateless) 주의 사항
```
인바운드: TCP 80 허용 ← 0.0.0.0/0   ← 설정
아웃바운드: TCP 1024-65535 허용 → 0.0.0.0/0  ← 이것도 별도로 설정 필요!
           (Ephemeral Port 범위 — HTTP 응답이 이 포트로 나감)
```

---

## 3. VM OS 방화벽 (Host-based Firewall)

클라우드 ACG를 통과한 트래픽을 **OS에서 한 번 더** 걸러내는 **최후 방어선**.

내부망 침입 시(다른 VM 탈취 후 수평 이동) ACG만으로는 막을 수 없음 → VM 자체 방화벽이 필수.

### Ubuntu / Debian — ufw

```bash
# 현재 상태 확인
sudo ufw status verbose

# 기본 정책 (권장: 인바운드 전부 차단, 아웃바운드 허용)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 포트 허용
sudo ufw allow 22/tcp       # SSH
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS

# 특정 IP만 SSH 허용 (권장)
sudo ufw allow from 10.0.1.5 to any port 22

# ufw 활성화
sudo ufw enable
sudo ufw status numbered

# 규칙 삭제
sudo ufw delete 3           # 번호로 삭제
```

### RHEL / CentOS / Rocky — firewall-cmd

```bash
# 상태 확인
sudo firewall-cmd --state
sudo firewall-cmd --list-all

# 포트 영구 허용 (--permanent 없으면 재부팅 시 사라짐)
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload

# 특정 IP만 22 허용 (rich rule)
sudo firewall-cmd --permanent --add-rich-rule='
  rule family=ipv4
  source address=10.0.1.5
  port port=22 protocol=tcp accept'
sudo firewall-cmd --reload

# zone 확인 (기본 zone: public)
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all --zone=public
```

> [!tip] `--permanent` 없이 테스트 후 `--permanent`로 영구 적용
> `--permanent` 없이 먼저 테스트 → 문제없으면 `--permanent` + `--reload`로 확정.

---

## 4. Defense in Depth — 다계층 보안 설계

```
인터넷 (제어 불가)
    ↓
Internet Gateway / VCN 경계
    ↓   L3 라우팅, 공인IP↔사설IP 변환
NSG / NACL  ← 서브넷 진입 전 1차 필터
    ↓   서브넷 단위, Stateless/Stateful
ACG / Security Group  ← VM 단위 2차 필터
    ↓   인스턴스 단위 세밀 제어, Stateful
VM Host Firewall  ← OS 레벨 최종 필터
    ↓   포트별 프로세스 제어
Application  ← 앱 자체 인증·검증
    JWT, API Key, WAF, 입력 검증
```

**계층이 많을수록 공격자가 뚫어야 하는 비용이 증가.** 단일 계층만 믿는 것은 금물.

### 각 계층의 역할 요약

| 계층 | 도구 | 방어 내용 |
|---|---|---|
| 네트워크 경계 | IGW, VCN 라우팅 | 공인IP 없는 Private Subnet은 인바운드 자체 차단 |
| 서브넷 | NSG / NACL | 서브넷 전체 진입 트래픽 1차 필터링 |
| 인스턴스 | ACG / SG | VM별 세밀한 포트·IP 제어 |
| OS | ufw / firewall-cmd | 내부망 수평 이동 차단, 마지막 방어선 |
| 애플리케이션 | JWT, OAuth2, WAF | 인증된 요청만 처리, SQL injection 등 방어 |

---

## 5. 자주 하는 실수 & 보안 체크리스트

### ❌ 자주 하는 실수
- `0.0.0.0/0` 으로 22(SSH) 포트 전체 오픈
- 개발 편의상 DB 포트(3306/5432)를 인터넷에 노출
- 테스트용 열어둔 포트를 프로덕션에 그대로 방치
- ACG만 설정하고 VM 방화벽 미설정
- Public/Private 서브넷 구분 없이 전부 Public에 배치

### ✅ 보안 체크리스트
- [ ] 서브넷이 역할별(Web/WAS/DB)로 분리되어 있는가?
- [ ] SSH/RDP는 특정 IP(Bastion/VPN)로만 제한?
- [ ] DB 포트가 WAS 서브넷 IP로만 제한되어 있는가?
- [ ] NSG + ACG 이중 설정 완료?
- [ ] VM OS 방화벽(ufw/firewall-cmd) 활성화 확인?
- [ ] 불필요한 오픈 포트 주기적 검토 중?

---

## 6. 관련
- [[VCN-VPC]] · [[Subnetting]] · [[HA-Architecture]] · [[Gateway]]
