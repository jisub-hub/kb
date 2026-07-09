---
tags:
  - database
  - deployment
  - architecture
  - decision
  - rds
  - patroni
  - k8s-operator
created: 2026-06-16
---

# DB 배포 방식 결정 트리

> [!summary] 한 줄 요약
> DB를 어디에 어떻게 올릴지는 팀 운영 역량·비용·SLA에 따라 결정한다. **관리형 > Patroni 직접운영 > K8s Operator** 순으로 운영 복잡도가 올라간다.

---

## 1. 결정 트리

```
운영팀에 DBA·K8s 전문가가 있는가?
  │
  ├─ NO  →  클라우드 관리형 서비스 (RDS, Cloud SQL, Atlas)
  │            └─ 가장 낮은 운영 부담, 가장 높은 비용
  │
  └─ YES  →  직접 운영 검토
               │
               ├─ K8s 환경인가?
               │    │
               │    ├─ YES + Operator 운영 역량 있음
               │    │    └─ K8s Operator (CNPG, Percona, Strimzi)
               │    │
               │    └─ YES + Operator 학습 부담이 큰 경우
               │         └─ K8s 밖 VM + Patroni (권장)
               │
               └─ VM/Bare-metal 환경
                    └─ Patroni + etcd + HAProxy
```

---

## 2. 옵션별 상세 비교

| 기준 | 관리형 (RDS 등) | VM + Patroni | K8s Operator |
|------|----------------|--------------|--------------|
| **운영 복잡도** | 낮음 | 중간 | 높음 |
| **Failover 시간** | 20~60초 (자동) | 10~30초 (자동) | 15~45초 (자동) |
| **비용** | 높음 (2~5× EC2 직접) | 중간 | 중간 + Operator 라이선스 |
| **커스터마이징** | 낮음 (파라미터 그룹) | 높음 | 중간 |
| **백업** | 자동 (PITR 지원) | 직접 설정 필요 | Operator가 처리 |
| **스케일 업** | 다운타임 최소화 | 수동 + 순단 | 롤링 업그레이드 가능 |
| **멀티 리전** | ✅ 기본 지원 | 직접 구성 필요 | ❌ 어려움 |
| **K8s 통합** | External Service | External Service | ✅ 네이티브 |
| **진입장벽** | 낮음 | Patroni 학습 필요 | Operator + K8s CRD 학습 |

---

## 3. 관리형 서비스 (Managed DB)

### 언제 선택하는가

```
✅ 추천 상황:
  - 팀에 DB 운영 전문가 없음
  - 빠른 런칭이 우선 (스타트업 초기)
  - 멀티 리전 복제 필요
  - 규정 준수 (SOC 2, HIPAA) 인증 필요
  - 운영 인력 비용이 인프라 비용보다 비쌈

❌ 피해야 할 상황:
  - 특수 PostgreSQL 확장 필요 (PostGIS, pg_cron 등 일부 미지원)
  - 비용 민감 대규모 워크로드
  - 온프레미스·에어갭 환경
```

### 주요 옵션

```
AWS RDS PostgreSQL      → 가장 성숙. Multi-AZ, Read Replica, Aurora 파생
Google Cloud SQL        → GCP 통합 편리. IAM 연동
Azure Database PG       → Azure AD 통합
Supabase                → PG 기반 BaaS, pgvector 기본 포함
Neon                    → Serverless PG (cold start 있음)
```

```yaml
# Spring Boot — RDS 연결 (SSL 필수)
spring:
  datasource:
    url: jdbc:postgresql://mydb.xxx.rds.amazonaws.com:5432/mydb
         ?sslmode=require&sslrootcert=/etc/ssl/rds-ca.pem
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      connection-timeout: 5000
```

---

## 4. VM + Patroni 직접 운영

### 언제 선택하는가

```
✅ 추천 상황:
  - 클라우드 비용 최소화 (관리형 대비 40~60% 절감)
  - 온프레미스 또는 전용 서버 환경
  - DB 운영 경험 있는 팀
  - PostgreSQL 확장(TimescaleDB, pgvector) 완전 제어 필요

❌ 피해야 할 상황:
  - 24/7 온콜 대응 인력 없음
  - 멀티 리전 DR 필요 (복잡도 급증)
```

### 아키텍처 (3노드)

```
  etcd 클러스터 (3노드, Raft 합의)
       │
  ┌────┴────┐
  │ Patroni │  ← 각 PostgreSQL 노드에 사이드카로 실행
  │  + PG   │    자동 Failover, pg_rewind, 슬롯 관리
  └────┬────┘
       │
  HAProxy ── 5432(쓰기/Primary) / 5433(읽기/Replica)
       │
  Spring Boot
  (읽기/쓰기 DataSource 분리)
```

→ 상세 설정은 [[PostgreSQL-HA-Cluster]] 참고

### 운영 필수 체크리스트

```bash
# Patroni 상태 확인
patronictl -c /etc/patroni/patroni.yml list

# etcd 클러스터 상태
etcdctl endpoint health --cluster

# 수동 Failover
patronictl -c /etc/patroni/patroni.yml failover --master pg-primary --candidate pg-replica-1

# 백업 (pgBackRest 권장)
pgbackrest --stanza=main backup --type=full
pgbackrest --stanza=main backup --type=diff

# PITR 복구
pgbackrest --stanza=main --delta restore \
  --target="2026-06-16 03:00:00" --target-action=promote
```

---

## 5. K8s Operator

### 언제 선택하는가

```
✅ 추천 상황:
  - 이미 K8s에서 모든 것을 운영 중 (플랫폼 단일화)
  - 개발팀이 K8s Operator 패턴에 익숙
  - 빠른 환경 복제 필요 (dev/stg/prod 동일 CRD)
  - GitOps (ArgoCD)로 DB 설정까지 코드화

❌ 피해야 할 상황:
  - K8s Operator를 처음 접하는 팀
  - 프로덕션 데이터 유실 리스크를 감당하기 어려운 경우
  - 노드 장애 시 PV 재연결 문제 해결 역량 없음
```

### 주요 Operator

| Operator | DB | 특징 |
|----------|-----|------|
| **CNPG** (CloudNativePG) | PostgreSQL | CNCF 샌드박스, 가장 성숙한 PG Operator |
| **Percona Operator** | PostgreSQL / MySQL / MongoDB | 멀티 DB 지원, 상업 지원 |
| **Strimzi** | Kafka | Kafka 전용 표준 Operator |
| **Redis Operator** | Redis | Spotahome/OT-container 등 여러 구현체 |

### CNPG 예시

```yaml
# PostgreSQL 클러스터 선언 (CRD)
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16

  storage:
    size: 100Gi
    storageClass: fast-ssd

  # Primary → Replica 동기 복제 (1 sync replica)
  minSyncReplicas: 1
  maxSyncReplicas: 1

  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      destinationPath: s3://my-bucket/pg-backup
      s3Credentials:
        accessKeyId:
          name: aws-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-creds
          key: SECRET_ACCESS_KEY

  monitoring:
    enablePodMonitor: true    # Prometheus PodMonitor 자동 생성
```

```bash
# CNPG 상태 확인
kubectl cnpg status pg-cluster

# 즉각 Failover
kubectl cnpg promote pg-cluster pg-cluster-2

# PITR 복구
kubectl cnpg restore pg-cluster --target-time "2026-06-16T03:00:00"
```

---

## 6. Redis 배포 옵션

| 옵션 | 설명 | 권장 상황 |
|------|------|-----------|
| **ElastiCache** | AWS 관리형 Redis | 클라우드, 빠른 설정 |
| **Redis Enterprise** | 상업용 클러스터 | 고성능, 엔터프라이즈 |
| **Docker + Sentinel** | VM 3노드 HA | 직접 운영, 중소규모 |
| **Redis Operator (K8s)** | CRD 기반 | K8s 네이티브 환경 |
| **Valkey** | Redis 오픈소스 포크 | 라이선스 회피, Redis 호환 |

```yaml
# Redis Sentinel 간단 구성 (docker-compose)
services:
  redis-primary:
    image: redis:7-alpine
    command: redis-server --appendonly yes

  redis-sentinel:
    image: redis:7-alpine
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/redis/sentinel.conf
```

---

## 7. 결론 요약

```
팀 역량이 없다 / 스타트업 초기 / 멀티리전 필요
  → 관리형 (RDS, Cloud SQL)

온프레미스 / 비용 최적화 / PG 완전 제어
  → VM + Patroni + etcd + HAProxy

K8s로 모든 것을 운영 / GitOps 지향 / 플랫폼팀 있음
  → K8s Operator (CNPG, Percona)

어떤 경우든 공통 원칙:
  ✅ 자동화된 백업 + PITR 필수
  ✅ 장애 복구 절차 문서화 + 주기적 복구 훈련
  ✅ 모니터링: pg_stat_activity, 슬로우쿼리, 복제 지연
  ✅ 연결 풀링: PgBouncer 또는 HikariCP 튜닝
```

---

## 8. 관련
- [[PostgreSQL-HA-Cluster]] — Patroni + etcd + HAProxy 상세 설정
- [[StatefulSet-DB]] — K8s에서 DB를 직접 올리면 안 되는 이유
- [[DB-Fundamentals]] — ACID, CAP, RDB vs NoSQL 기초
- [[../k8s/GitOps]] — ArgoCD로 CNPG CRD GitOps 적용
