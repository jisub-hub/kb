---
tags:
  - network
  - ha
  - high-availability
  - failover
  - architecture
created: 2026-06-15
---

# HA 아키텍처 — 단일화 vs 이중화 & Cross-Connected

> [!summary] 한 줄 요약
> 단일 서버 구성(Single Zone)은 장애 시 **SPOF(단일 장애점)**가 된다. **Multi-AD/AZ 이중화 + Cross-Connected 서비스 계층**으로 단일 장애점을 없애야 무중단 서비스가 가능하다.

---

## 1. 단일화 구조 (Single Zone) — 문제점

```
인터넷
  └→ IGW
      └→ Web Server  ←── SPOF
              └→ DB Server  ←── SPOF
```

| 문제 | 영향 |
|---|---|
| 서버/존 장애 | **전체 서비스 중단** |
| 배포 | 서비스 중단 필요 (다운타임 발생) |
| 트래픽 급증 | Scale-out 불가 → 과부하 |

---

## 2. 이중화 구조 (Multi-AD / HA) — 기본

```
인터넷
  └→ Load Balancer (HA 구성)
        ├→ Web AD-1
        └→ Web AD-2
              ├→ DB Primary (AD-1)
              └→ DB Standby (AD-2) ←── 복제
```

**이중화 구성의 핵심 원칙:**
- 모든 레이어(LB, Web, WAS, DB)를 **가용 영역(AD/AZ) 별로 최소 2개** 배치
- DB는 **Primary-Standby 복제** → Primary 장애 시 자동 Failover
- LB 자체도 클라우드 관리형 사용 시 자동 HA (OCI LB, AWS ALB 등은 기본 HA)

**장점:**
| 항목 | 효과 |
|---|---|
| 가용성 | 한 AD/AZ 장애 시 자동 Failover (무중단) |
| 배포 | Rolling Deploy → 다운타임 없는 배포 |
| 확장 | Scale-out으로 트래픽 분산 처리 |

---

## 3. 서비스 계층 이중화 방식 비교

### Siloed (단일경로) 이중화 — 문제 있음

```
Load Balancer
  ├→ web-1 → was-1  ✕ (장애)
  └→ web-2 → was-2  ✓
```

**was-1 장애 시:**
1. LB는 web-1이 살아있어 계속 트래픽 전송
2. web-1이 was-1 접속 실패 → **500/502 오류 반환**
3. 수동으로 web-1 설정을 was-2로 변경해야 복구

→ **이중화했는데 실질적으로 SPOF와 같은 결과**

---

### Cross-Connected 이중화 — 권장

```
Load Balancer
  ├→ web-1 ─┐
  └→ web-2 ─┤
             ├→ was-1  ✕ (장애)
             └→ was-2  ✓  ← web-1, web-2 모두 was-2로 전환
```

**was-1 장애 시:**
1. web 서버가 was Health Check 감지 → was-1을 풀에서 제거
2. web-1, web-2 모두 was-2로 요청 → **자동 Failover**
3. LB 개입 없이 web 레벨에서 처리

**구현 방법:**
- Nginx upstream에 was-1, was-2 모두 등록
- `upstream was { server was-1:8080; server was-2:8080; }`
- Nginx가 자체 Health Check로 장애 서버 자동 제외

---

## 4. DB 이중화 (Primary-Standby)

```
WAS → DB Primary (AD-1)  ──복제──→  DB Standby (AD-2)
              ↓ 장애 발생
              Failover 자동 실행
WAS → DB Standby가 Primary로 승격 (AD-2)
```

| 방식 | 동기화 | 특징 |
|---|---|---|
| **동기 복제** | Primary 커밋 = Standby 적용 완료 | RPO=0, 지연 있음 |
| **비동기 복제** | Primary 커밋 후 나중에 Standby 반영 | 지연 없음, 일부 데이터 손실 가능 |
| **Semi-sync** | 최소 1개 Standby 확인 후 커밋 | 균형점 |

**클라우드 관리형 DB HA:**
| 클라우드 | 서비스 |
|---|---|
| OCI | Autonomous DB, MySQL HeatWave (자동 HA) |
| AWS | RDS Multi-AZ, Aurora |
| Azure | Azure Database (Zone-redundant HA) |

---

## 5. RTO / RPO 개념

| 지표 | 의미 | 목표 |
|---|---|---|
| **RTO** (Recovery Time Objective) | 장애 발생 후 서비스 복구까지 허용 시간 | 짧을수록 좋음 |
| **RPO** (Recovery Point Objective) | 복구 시점까지 허용하는 데이터 손실 범위 | 0에 가까울수록 좋음 |

- **Active-Standby**: RTO 수 초~분, RPO 동기복제면 0
- **Active-Active**: RTO ≈ 0, RPO ≈ 0 (가장 이상적, 구성 복잡)

---

## 6. 배포 전략과 이중화

이중화 구성이 있어야 무중단 배포가 가능하다.

| 전략 | 방식 | 다운타임 |
|---|---|---|
| **Rolling Deploy** | 서버 1대씩 순차 교체 | 없음 (이중화 필요) |
| **Blue-Green** | 신구 환경을 동시 유지, LB에서 전환 | 없음 (트래픽 즉시 전환) |
| **Canary** | 신 버전을 일부 트래픽에만 점진적 노출 | 없음 (이상 시 즉시 롤백) |

---

## 7. 관련
- [[VCN-VPC]] · [[Load-Balancer]] · [[Security-ACG-NSG]] · [[Gateway]]
