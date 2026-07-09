---
tags:
  - infra
  - kubernetes
  - postgresql
  - cnpg
  - database
  - operator
created: 2026-06-17
---

# CloudNativePG (CNPG) — K8s PostgreSQL 오퍼레이터

> [!summary] 한 줄 요약
> Kubernetes에서 PostgreSQL을 **Operator + Cluster CRD**로 선언적으로 운영하는 표준 도구. 자동 페일오버·백업/PITR·롤링 업그레이드·모니터링을 K8s 네이티브로 제공한다. "DB를 K8s에 올릴지" 결정은 [[DB-Deployment-Decision]], "왜 StatefulSet 직접은 힘든가"는 [[StatefulSet-DB]] 참고 — 이 노트는 **CNPG 운영 실무**다.

---

## 1. 왜 오퍼레이터인가

```
StatefulSet 직접 운영의 한계 (→ [[StatefulSet-DB]]):
  페일오버·복제·백업·PITR을 전부 수동 스크립트로 → 운영 부담·실수 위험

CNPG (Operator 패턴):
  Cluster CRD에 "원하는 상태"를 선언 → 오퍼레이터가 조정(reconcile)
  primary 장애 → 자동 replica 승격, WAL 백업, PITR, 롤링 업그레이드 자동화
  → 운영 지식을 오퍼레이터가 코드로 담음
```

CNPG는 EDB가 주도하는 CNCF 프로젝트로, **Patroni/Stolon 없이** 자체 instance manager로 HA를 구현한다(외부 합의 시스템 의존 최소화).

---

## 2. 아키텍처

```
┌─ CNPG Operator (Deployment) ── Cluster CRD watch·reconcile
│
└─ Cluster "mycluster"
     ├─ Pod: primary (read-write)        ┐
     ├─ Pod: replica-1 (streaming repl)  ├─ 각 Pod에 instance manager(PID 1)
     └─ Pod: replica-2 (read-only)       ┘
     + PVC(각 인스턴스 전용 스토리지)
     + Service: -rw(primary) / -ro(replica) / -r(any)
```

- **Instance Manager**: 각 Pod의 PID 1. PostgreSQL 생명주기·복제·헬스를 오퍼레이터와 협조해 관리.
- **합의**: K8s API(Cluster status)를 진실원천으로 사용 → 별도 etcd/ZooKeeper 불필요.

---

## 3. Cluster 정의 (선언적)

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: mycluster
spec:
  instances: 3                      # primary 1 + replica 2
  imageName: ghcr.io/cloudnative-pg/postgresql:16
  storage:
    size: 50Gi
    storageClass: fast-ssd          # Retain 권장 (→ [[PV-PVC]])
  postgresql:
    parameters:                     # postgresql.conf
      max_connections: "200"
      shared_buffers: "512MB"
  bootstrap:
    initdb:
      database: appdb
      owner: appuser
  monitoring:
    enablePodMonitor: true          # Prometheus 연동
```

> 인스턴스 수·스토리지·PG 파라미터를 YAML로 선언 → 오퍼레이터가 Pod·PVC·Service를 만든다.

---

## 4. 고가용성 & 페일오버

```
정상:   primary(rw) ──streaming replication──► replica(ro)
장애:   primary Pod/노드 장애 감지
        → 가장 최신 replica를 자동 승격(promote)
        → -rw Service 엔드포인트를 새 primary로 전환
        → 구 primary 복구 시 replica로 재합류

switchover: 계획된 전환(유지보수) — kubectl cnpg promote 로 수동 지정
RPO: 동기 복제(synchronous) 설정 시 0에 근접, 비동기는 약간의 손실 가능
```

```yaml
# 동기 복제(데이터 손실 최소화)
spec:
  postgresql:
    synchronous:
      method: any
      number: 1            # 최소 1개 replica가 동기 확인
```

> 애플리케이션은 `-rw` Service(쓰기), `-ro` Service(읽기 분산)로 접속 → 페일오버 시 커넥션만 재연결하면 됨.

---

## 5. 백업 & 복구 (PITR)

CNPG는 Barman 기반으로 **베이스 백업 + WAL 아카이빙**을 object store(S3 등)에 저장한다.

```yaml
spec:
  backup:
    barmanObjectStore:
      destinationPath: s3://my-bucket/cnpg
      s3Credentials:
        accessKeyId: { name: s3-creds, key: ACCESS_KEY_ID }
        secretAccessKey: { name: s3-creds, key: SECRET_ACCESS_KEY }
      wal:
        compression: gzip
    retentionPolicy: "30d"
---
# 스케줄 백업
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata: { name: daily }
spec:
  schedule: "0 3 * * *"            # 매일 03:00
  cluster: { name: mycluster }
```

```
PITR(Point-In-Time Recovery): 베이스 백업 + WAL 재생으로 특정 시점 복원
  → 새 Cluster를 bootstrap.recovery로 생성, recoveryTarget(시각/LSN) 지정
실수 삭제·논리 오류 복구의 핵심. 백업만 있고 복구 리허설 안 하면 무의미 → 정기 복구 훈련.
```

---

## 6. 모니터링

```
enablePodMonitor: true → Prometheus가 각 인스턴스 메트릭 수집
주요 메트릭:
  cnpg_pg_replication_lag       # 복제 지연 (페일오버 안전성)
  cnpg_pg_database_size_bytes   # DB 크기
  cnpg_backends_total           # 연결 수
  cnpg_pg_wal_archive_status    # WAL 아카이빙 상태(백업 건전성)
  cnpg_collector_up             # 인스턴스 헬스
```

→ [[Prometheus-Grafana]]로 대시보드. CNPG 공식 Grafana 대시보드 제공. 복제 지연·WAL 아카이빙 실패는 즉시 알림 대상.

---

## 7. 업그레이드 & 유지보수

```
오퍼레이터 업그레이드: Deployment 이미지 교체 (롤링)
PostgreSQL minor 업그레이드: imageName 변경 → 오퍼레이터가 replica부터 롤링,
                            마지막에 switchover로 primary 교체 (무중단에 가까움)
PostgreSQL major 업그레이드: 별도 절차(논리 복제/덤프) — minor보다 신중
인스턴스 스케일: spec.instances 변경 → replica 추가/제거
```

> 롤링 업그레이드는 replica → primary 순. switchover 순간 짧은 쓰기 중단 가능 → 커넥션 풀 재시도([[Resilience4j]]) 권장.

---

## 8. 연결 & 커넥션 풀

```
Service:
  mycluster-rw  → primary (읽기·쓰기)
  mycluster-ro  → replica (읽기 전용 분산)
  mycluster-r   → 아무 인스턴스 (읽기)

커넥션 풀: CNPG는 PgBouncer 기반 Pooler CRD 제공
  → 다수 애플리케이션 연결을 풀링 (→ [[../../programming/spring/data/PgBouncer]])
```

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata: { name: mycluster-pooler }
spec:
  cluster: { name: mycluster }
  instances: 2
  type: rw
  pgbouncer:
    poolMode: transaction
```

---

## 9. 운영 체크리스트

```
□ storageClass는 Retain (PVC 삭제돼도 데이터 보존)
□ 동기 복제 여부 결정 (RPO 0 필요 vs 성능)
□ ScheduledBackup + object store + retention 설정
□ PITR 복구 리허설 정기 수행 (백업≠복구 보장)
□ 복제 지연·WAL 아카이빙 메트릭 알림
□ Pooler(PgBouncer)로 연결 폭주 방어
□ 노드 분산(anti-affinity) — primary·replica를 다른 노드에
□ 리소스 requests/limits 명시 (→ [[../cloud/Cost-Optimization]])
```

---

## 10. 정리

> CNPG = **PostgreSQL HA·백업·업그레이드 운영 지식을 K8s Operator로 코드화**한 것. Cluster CRD 선언 → 자동 페일오버·PITR·롤링 업그레이드. Patroni 직접 운영 대비 부담이 크게 준다. 단 **PITR 복구 리허설·복제 지연 모니터링**은 사람이 챙겨야 한다.

---

## 관련
- [[StatefulSet-DB]] — K8s에서 DB 직접 운영의 어려움(이 오퍼레이터가 푸는 문제)
- [[DB-Deployment-Decision]] — Managed RDS vs Patroni vs Operator 결정
- [[PostgreSQL-HA-Cluster]] — PostgreSQL HA 일반 (Patroni 등)
- [[PV-PVC]] · [[Prometheus-Grafana]] · [[../../programming/spring/data/PgBouncer]]
