---
tags:
  - kubernetes
  - k8s
  - database
  - statefulset
  - redis
  - antipattern
created: 2026-06-16
---

# K8s에서 DB·Redis를 Pod로 운영하지 않는 이유

> [!summary] 한 줄 요약
> DB·Redis를 K8s Pod로 운영하면 **StatefulSet 설정 지옥 + 데이터 안전성 문제**가 발생한다. 프로덕션에서는 관리형 서비스(RDS, ElastiCache) 또는 전용 VM을 쓰고, K8s는 **무상태(Stateless) 앱 레이어만** 오케스트레이션한다.

---

## 1. 왜 K8s에서 DB를 Pod로 운영하면 안 되나

```
[Pod의 기본 특성]
  - Pod는 언제든 재시작·이동 가능 (Ephemeral)
  - 노드 장애 시 K8s Scheduler가 다른 노드에 재배치
  - Rolling Update 시 새 Pod 생성 → 구 Pod 삭제

[DB·Redis는 상태를 가짐]
  - 데이터가 디스크에 있어야 함
  - 재시작 시 동일 데이터 유지 필요
  - 주소(IP)가 바뀌면 복제 설정 재구성 필요
  - 클러스터 구성(Primary/Replica, 쿼럼)이 깨지면 안 됨

[충돌의 결과]
  Pod 재배치 → 마운트된 PVC도 이동? → 노드에 따라 불가
  Pod 재시작 → Redis AOF/RDB 파일 손상 위험
  Rolling Update → DB 다운타임 발생 (순서 보장 필요)
  노드 OOM → DB Pod Kill First (QoS 정책 없으면)
```

---

## 2. StatefulSet이란?

`StatefulSet`은 DB처럼 **상태를 가진 Pod**를 위한 K8s 워크로드 오브젝트. Deployment와 달리 각 Pod에 안정적인 이름·네트워크·저장소를 부여한다.

```
Deployment (무상태)         StatefulSet (유상태)
───────────────────         ─────────────────────────
Pod-7f3a8                   mysql-0
Pod-9b2c1       ←무작위     mysql-1   ←순번 고정
Pod-4d1e9                   mysql-2

Pod 재시작 시 IP 변경        Pod 재시작 시 같은 이름 유지
PVC 공유 불가                각 Pod에 개별 PVC 자동 생성
순서 무관 생성               0 → 1 → 2 순서대로 생성
                             2 → 1 → 0 순서대로 삭제
```

### StatefulSet 기본 구조

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"         # Headless Service 이름
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2"

  # Pod마다 개별 PVC 자동 생성
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"    # 반드시 빠른 스토리지 클래스
      resources:
        requests:
          storage: 100Gi

---
# Headless Service: DNS로 각 Pod에 직접 접근
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None              # Headless
  selector:
    app: mysql
  ports:
  - port: 3306

# DNS: mysql-0.mysql.namespace.svc.cluster.local
#      mysql-1.mysql.namespace.svc.cluster.local
```

---

## 3. StatefulSet의 설정 지옥

```
① 스토리지 프로비저닝
   ReadWriteOnce PVC → Pod가 올라간 노드에만 마운트 가능
   Pod 재배치 = 스토리지도 같은 노드여야 함 → 스케줄링 제한
   클라우드 EBS(AWS): 가용영역 고정 → Zone 장애 = DB 장애

② 네트워크 ID 안정성
   StatefulSet은 안정적 DNS 제공하지만 IP는 여전히 바뀜
   MySQL Group Replication, Redis Cluster는 IP로 구성 정보 저장
   → Pod 재시작 마다 클러스터 재구성 스크립트 필요

③ 데이터 마이그레이션
   DB 버전 업그레이드 = StatefulSet 이미지 변경
   → 롤링 업데이트가 데이터 포맷 변환 실행 중 절반씩 뒤섞임
   → 수동 개입 필요

④ 백업
   PVC 스냅샷 타이밍: DB flush + fsync + PVC snapshot 순서 보장 필요
   K8s VolumeSnapshot API는 DB-aware하지 않음
   → pg_dump, xtrabackup 같은 별도 백업 Job 필요

⑤ 운영자(Operator) 없으면 불가
   PostgreSQL: CloudNativePG, Crunchy Postgres Operator
   MySQL: MySQL Operator, Percona Operator
   Redis: Redis Operator (Spotahome), Redis Enterprise Operator
   
   → Operator 없이 직접 하면 Failover, 복구, 백업 모두 수동
   → Operator = 별도 학습 곡선 + 버전 종속성

⑥ 리소스 경합
   DB와 앱이 같은 클러스터 → DB OOM 시 K8s가 Pod Kill
   OOM Killer 우선순위: QoS Guaranteed > Burstable > BestEffort
   DB를 Guaranteed로 설정하려면 requests = limits (과할당 불가)
```

---

## 4. 프로덕션 권장 아키텍처

```
[추천]
  애플리케이션 레이어 (Spring, FastAPI)
    → K8s Deployment/HPA로 수평 확장
    → Stateless, 얼마든지 재배치 가능

  데이터 레이어 (PostgreSQL, Redis, Kafka)
    → 관리형 서비스 (AWS RDS, ElastiCache, MSK)
      : 자동 백업, Failover, 패치, 모니터링 포함
    → 또는 전용 VM + Patroni/etcd/HAProxy
      : K8s 클러스터 장애가 DB에 영향 없음

┌─────────────────────────────────┐
│  K8s Cluster                    │
│  ┌─────────┐  ┌─────────────┐  │
│  │  App    │  │  API GW     │  │
│  │ (Pods)  │  │ (Ingress)   │  │
│  └────┬────┘  └─────────────┘  │
└───────┼─────────────────────────┘
        │ (외부 연결)
┌───────┴──────────────────────────┐
│  Managed Services / Dedicated VM │
│  RDS PostgreSQL  ElastiCache Redis│
│  MSK Kafka       OpenSearch       │
└──────────────────────────────────┘
```

---

## 5. K8s에서 DB를 써도 되는 경우

```
✅ 개발/테스트 환경
   - 데이터 손실 허용
   - 빠른 초기화/삭제 필요
   - 실제 DB 설정 테스트 (Testcontainers가 더 낫지만)

✅ 소규모 내부 툴
   - 트래픽·데이터 소량
   - 관리형 서비스 비용이 문제
   - 데이터 손실 허용 범위 넓음

✅ 충분한 Operator + 팀 역량
   CloudNativePG, Vitess(MySQL), Redis Enterprise Operator
   + K8s 스토리지 전문가 + DBA가 팀에 있을 때

❌ 피해야 할 경우 (프로덕션 중요 데이터)
   - 금융, 주문, 사용자 계정 데이터
   - SLA가 있는 서비스
   - DBA 없이 운영하는 팀
```

---

## 6. 관련
- [[PV-PVC]] — PersistentVolume, StorageClass, 동적 프로비저닝
- [[../../infra/database/PostgreSQL-HA-Cluster]] — VM 기반 Patroni + etcd + HAProxy
- [[../../infra/database/DB-Fundamentals]] — RDB vs NoSQL, ACID
- [[Pod]] — QoS, Resources, OOM Kill 우선순위
