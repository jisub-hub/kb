---
tags:
  - database
  - postgresql
  - ha
  - patroni
  - haproxy
  - etcd
  - clustering
created: 2026-06-16
---

# PostgreSQL 수동 HA 클러스터 — Patroni + etcd + HAProxy

> [!summary] 한 줄 요약
> **Patroni**가 PostgreSQL 자동 장애조치(Failover)를 담당하고, **etcd**가 리더 선출·클러스터 상태 합의를 제공하며, **HAProxy**가 읽기/쓰기 트래픽을 올바른 노드로 라우팅한다. 세 컴포넌트가 맞물려 PostgreSQL 수동 HA 클러스터의 표준 스택.

---

## 1. 전체 아키텍처

```
[애플리케이션]
     │
     │  (단일 엔드포인트)
     ▼
[HAProxy]                         ← 트래픽 라우팅
  5432 → Primary만 (쓰기)
  5433 → 모든 노드 (읽기 분산)
     │
     ├──────────────────────────┐
     ▼                          ▼
[PG Node 1 - Primary]      [PG Node 2 - Standby]   [PG Node 3 - Standby]
  Patroni Agent                Patroni Agent           Patroni Agent
  PostgreSQL                   PostgreSQL              PostgreSQL
     │  DCS 쓰기                    │  DCS 읽기              │
     └──────────────────────────────┴────────────────────────┘
                                    │
                              [etcd Cluster]         ← 리더 선출, 상태 저장
                               etcd-1  etcd-2  etcd-3
                               (최소 3노드, 과반수 합의)

장애조치 흐름:
  Node1 장애 감지
  → Patroni가 etcd에서 새 리더 선출 (< 30초)
  → Node2가 Primary로 승격
  → Patroni가 etcd에 새 Primary 등록
  → HAProxy가 헬스체크 통해 Node2로 쓰기 트래픽 전환
  → 애플리케이션: 같은 HAProxy 주소 그대로 사용 (투명)
```

---

## 2. etcd — 분산 합의 저장소

```
역할:
  - Patroni 클러스터 상태 저장 (누가 Primary인지)
  - 리더 선출 (Raft 알고리즘, 과반수 필요)
  - 설정·Lock 저장

etcd 키 구조 (Patroni):
  /service/{cluster-name}/leader        ← 현재 Primary 호스트명
  /service/{cluster-name}/members/node1 ← 각 노드 상태 (역할, 엔드포인트, xlog_location)
  /service/{cluster-name}/config        ← 클러스터 설정 (동적 변경 가능)
  /service/{cluster-name}/history       ← 장애조치 이력

최소 구성: 3노드 (1대 장애 시 2/3 과반수 유지)
권장 구성: 5노드 (2대 동시 장애 허용)
```

### etcd 설치 및 클러스터 구성

```bash
# etcd 설치 (각 노드)
ETCD_VER=v3.5.12
curl -L https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  | tar xzf - -C /usr/local/bin --strip-components=1

# etcd1 (192.168.1.11), etcd2 (192.168.1.12), etcd3 (192.168.1.13)
# /etc/etcd/etcd.conf.yml (etcd1 예시)

cat > /etc/etcd/etcd.conf.yml <<EOF
name: etcd1
data-dir: /var/lib/etcd
listen-client-urls: http://0.0.0.0:2379
advertise-client-urls: http://192.168.1.11:2379
listen-peer-urls: http://0.0.0.0:2380
initial-advertise-peer-urls: http://192.168.1.11:2380
initial-cluster: etcd1=http://192.168.1.11:2380,etcd2=http://192.168.1.12:2380,etcd3=http://192.168.1.13:2380
initial-cluster-state: new
initial-cluster-token: pg-cluster-token
EOF

# Systemd 서비스
cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
After=network.target

[Service]
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.conf.yml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl enable --now etcd

# 클러스터 상태 확인
etcdctl --endpoints=http://192.168.1.11:2379,http://192.168.1.12:2379,http://192.168.1.13:2379 \
  member list
```

---

## 3. Patroni — PostgreSQL 자동 HA 에이전트

### 설치

```bash
# pip으로 설치 (각 PG 노드)
pip install patroni[etcd3]

# PostgreSQL도 설치 (각 노드)
apt install postgresql-16
```

### Patroni 설정 (`/etc/patroni/patroni.yml`)

```yaml
# pg-node1 (192.168.1.21) 예시
scope: pg-cluster      # etcd에 저장되는 클러스터 이름
namespace: /service/
name: pg-node1         # 이 노드의 이름 (노드마다 다름)

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.21:8008

etcd3:
  hosts:
    - 192.168.1.11:2379
    - 192.168.1.12:2379
    - 192.168.1.13:2379

bootstrap:
  dcs:
    ttl: 30                          # Primary heartbeat TTL (초)
    loop_wait: 10                    # 헬스체크 주기 (초)
    retry_timeout: 10
    maximum_lag_on_failover: 1048576 # 최대 Standby 지연 (1MB 이상 지연 시 failover 후보 제외)
    postgresql:
      use_pg_rewind: true            # Failover 후 구 Primary 복구 시 pg_rewind 사용
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        wal_keep_size: 1GB
        archive_mode: "on"
        archive_command: "cp %p /var/lib/postgresql/wal_archive/%f"

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 192.168.1.0/24 md5
    - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: admin_password
      options: [createrole, createdb]
    replicator:
      password: replicator_password
      options: [replication]

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.21:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: replicator_password
    superuser:
      username: postgres
      password: postgres_password

tags:
  nofailover: false        # true이면 이 노드는 Primary 후보 제외
  noloadbalance: false     # true이면 HAProxy 읽기 분산에서 제외
  clonefrom: false
  nosync: false
```

```bash
# Patroni 서비스
cat > /etc/systemd/system/patroni.service <<EOF
[Unit]
Description=Patroni HA PostgreSQL
After=network.target etcd.service

[Service]
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP \$MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now patroni

# 클러스터 상태 확인
patronictl -c /etc/patroni/patroni.yml list

# 출력 예:
# + Cluster: pg-cluster ----+----+-----------+
# | Member   | Host         | Role    | State   | TL | Lag in MB |
# +---------+--------------+---------+---------+----+-----------+
# | pg-node1 | 192.168.1.21 | Leader  | running |  1 |           |
# | pg-node2 | 192.168.1.22 | Replica | running |  1 |         0 |
# | pg-node3 | 192.168.1.23 | Replica | running |  1 |         0 |
```

---

## 4. HAProxy — 트래픽 라우팅

HAProxy는 Patroni REST API(`/primary`, `/replica`)를 헬스체크로 활용해 **현재 Primary/Replica를 자동으로 구분**한다.

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 4096
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect  5s
    timeout client  60s
    timeout server  60s

#──────────────────────────────────────────────────────────────────
# 쓰기 포트 (5432) — Primary만
#──────────────────────────────────────────────────────────────────
frontend pg_write
    bind *:5432
    default_backend pg_primary

backend pg_primary
    option httpchk GET /primary        # Patroni REST API: Primary이면 200, 아니면 503
    http-check expect status 200
    default-server inter 3s fastinter 1s fall 3 rise 2
    server pg-node1 192.168.1.21:5432 check port 8008
    server pg-node2 192.168.1.22:5432 check port 8008
    server pg-node3 192.168.1.23:5432 check port 8008

#──────────────────────────────────────────────────────────────────
# 읽기 포트 (5433) — 모든 Replica에 분산
#──────────────────────────────────────────────────────────────────
frontend pg_read
    bind *:5433
    default_backend pg_replicas

backend pg_replicas
    balance roundrobin
    option httpchk GET /replica        # Replica이면 200
    http-check expect status 200
    default-server inter 3s fastinter 1s fall 3 rise 2
    server pg-node1 192.168.1.21:5432 check port 8008
    server pg-node2 192.168.1.22:5432 check port 8008
    server pg-node3 192.168.1.23:5432 check port 8008

#──────────────────────────────────────────────────────────────────
# HAProxy Stats 페이지
#──────────────────────────────────────────────────────────────────
frontend stats
    bind *:7000
    stats enable
    stats uri /
    stats refresh 5s
    stats auth admin:admin
```

```bash
# HAProxy 재시작 (무중단 리로드)
systemctl reload haproxy
# http://haproxy-ip:7000 — 실시간 연결 현황
```

---

## 5. 장애조치 시나리오

### Primary 장애 → 자동 Failover

```
1. pg-node1 (Primary) 장애 발생
2. Patroni (pg-node2, pg-node3): Primary heartbeat TTL(30초) 초과 감지
3. etcd에서 리더 Lock 만료 → pg-node2가 Lock 획득
4. pg-node2 Patroni: PostgreSQL을 Primary로 승격 (pg_ctl promote)
5. etcd /service/pg-cluster/leader = "pg-node2" 업데이트
6. HAProxy: GET /primary → pg-node2는 200, pg-node3는 503
   → 쓰기 트래픽 자동으로 pg-node2로 라우팅
7. pg-node1 복구 후:
   - pg_rewind로 구 Primary를 새 Primary 기준으로 동기화
   - Replica로 재합류
```

```bash
# 수동 Failover (계획된 유지보수)
patronictl -c /etc/patroni/patroni.yml failover pg-cluster \
  --master pg-node1 \
  --candidate pg-node2 \
  --force

# Switchover (Zero 다운타임 Primary 변경)
patronictl -c /etc/patroni/patroni.yml switchover pg-cluster \
  --master pg-node1 \
  --candidate pg-node2

# 일시 정지 (자동 Failover 비활성화 — 유지보수 시)
patronictl -c /etc/patroni/patroni.yml pause pg-cluster
patronictl -c /etc/patroni/patroni.yml resume pg-cluster
```

### Split-Brain 방지

```
etcd 과반수가 없는 경우:
  - Patroni: DCS 접근 불가 → Primary 역할 포기 (안전 모드)
  - 쓰기 트래픽: HAProxy 헬스체크 실패 → 전체 502
  - 의도적 설계: 데이터 불일치보다 서비스 중단이 낫다 (CP 우선)

etcd 3노드 중 1노드 장애:
  - 2/3 과반수 유지 → 정상 동작 계속
```

---

## 6. 페일오버 타이밍 & 네트워크 분단

> §5는 "어떻게 페일오버되는가"를 다뤘다. 여기선 **"몇 초 걸리는가(RTO)"와 "어디서 데이터를 잃거나 두 Primary가 생기는가"** 를 수치·시나리오로 본다. SLA를 약속하려면 이 단계별 시간 합을 알아야 한다.

### 6.1. 페일오버 시간 분해 (RTO 추정)

자동 페일오버 RTO는 단일 값이 아니라 **순차 단계의 합**이다. §3 설정 기준(`ttl: 30`, `loop_wait: 10`, `retry_timeout: 10`):

```
T0  Primary(node1) 다운
│
├─ (1) 장애 감지 지연 ──────────── 최대 loop_wait (10초)
│      Standby Patroni는 loop_wait 주기로 헬스체크/리더 갱신 확인.
│      마지막 체크 직후 죽으면 다음 루프까지 못 본다 → 0~10초 변동.
│
├─ (2) 리더 임차(lease) 만료 ───── 최대 ttl (30초)
│      etcd의 leader 키는 TTL=30s. Primary가 죽으면 더 이상 갱신 못 함
│      → TTL 만료까지 기다려야 Lock이 풀린다. 이게 보통 RTO의 최대 비중.
│      (이미 (1)에서 흐른 시간과 겹침: 실제론 max(잔여 TTL) 관점)
│
├─ (3) 리더 선출 + 합의 ────────── 수백 ms ~ 수초
│      가장 앞선(lag 최소) Standby가 etcd에 새 leader 키 CAS 획득(Raft 합의).
│      maximum_lag_on_failover 초과 노드는 후보 제외.
│
├─ (4) PostgreSQL 승격 ─────────── 1~5초
│      pg_ctl promote: WAL 재생 마무리, read-write 전환.
│
├─ (5) HAProxy 재구성 ──────────── inter(3s) × fall(3) = 최대 ~9초
│      httpchk GET /primary 가 새 노드에서 200으로 바뀌는 걸 감지.
│      구 Primary는 fall 횟수만큼 실패해야 backend에서 빠진다.
│
▼
총 RTO ≈ ttl(30) + 승격(~3) + HAProxy 전환(~6~9) ≈ 40~45초 (위 기본값 기준)
        ※ (1)·(2)는 상당 부분 겹쳐 흐르므로 ttl이 지배항.
```

- **체감 RTO를 줄이려면**: `ttl`을 낮추는 게 가장 효과가 크다(아래 6.3 트레이드오프). HAProxy `inter`/`fall`도 줄이면 전환 감지가 빨라진다.
- **애플리케이션 측**: 커넥션 풀은 죽은 Primary 커넥션을 잡고 있다가 타임아웃 → 풀의 `validation`/`maxLifetime`과 retry가 없으면 체감 다운타임이 RTO보다 길어진다.

### 6.2. Split-Brain (두 Primary)

가장 위험한 장애. **네트워크 분단 시 두 노드가 동시에 Primary가 되어 양쪽에 쓰기가 들어가면 데이터가 분기(divergence)** 한다 — 이후 자동 화해 불가, 수동으로 한쪽 데이터를 버려야 한다.

```
시나리오: node1(Primary)이 etcd와의 네트워크만 끊김 (PG 자체는 살아 있음)

  [올바른 동작 — watchdog/fencing 있음]
  node1: TTL 내에 leader 키 갱신 실패 임박
         → ttl 만료 전에 스스로 read-only로 강등(demote) 후 종료
         → watchdog 타이머가 demote 실패 시 OS를 강제 리부트(fence)
  node2: leader Lock 획득 → 승격
  결과: Primary는 항상 하나. 안전.

  [잘못된 동작 — 타임아웃 설정 오류 / watchdog 없음]
  node1: 자신이 아직 Primary라 착각하고 쓰기 계속 수신
  node2: leader Lock 만료로 자신도 승격 → 쓰기 수신
  결과: 두 Primary 동시 쓰기 → 데이터 분기 → 복구 시 한쪽 트랜잭션 영구 손실
```

- **방지 메커니즘**:
  - **watchdog**(softdog 등): Patroni가 주기적으로 watchdog을 "쓰다듬는다(pet)". 데모트/갱신에 실패하면 watchdog이 OS를 강제 리부트 → 좀비 Primary 제거. **프로덕션 필수.**
  - **fencing**: 분단된 노드를 네트워크/전원 차원에서 격리(STONITH 사상).
  - Patroni는 기본적으로 **`ttl > loop_wait + retry_timeout`** 관계를 지키면 단일 노드가 안전하게 강등될 시간을 확보한다 — 이 관계가 깨지면 split-brain 창이 열린다(6.3).

### 6.3. 파라미터 튜닝 트레이드오프

`loop_wait` / `ttl` / `retry_timeout`은 **RTO/RPO ↔ 오탐(false failover)** 사이의 다이얼이다.

| 파라미터 | 의미 | 짧게 (예: ttl=15) | 길게 (예: ttl=60) |
|----------|------|-------------------|-------------------|
| `ttl` | 리더 임차 유효시간 | RTO↓ (빠른 페일오버) | RTO↑, 오탐↓ |
| `loop_wait` | 헬스체크 주기 | 감지 빠름, etcd 부하↑ | 감지 느림 |
| `retry_timeout` | DCS/PG 작업 재시도 한계 | 일시 지연을 장애로 오인↑ | 끈기 있게 버팀 |

```
짧은 타임아웃의 위험 — false failover:
  GC stop-the-world, 디스크 IO 스파이크, 짧은 네트워크 흔들림(수 초)
  → ttl 안에 leader 키 갱신을 못 함 → 멀쩡한 Primary가 강등/페일오버됨
  → 불필요한 다운타임 + 커넥션 끊김 + pg_rewind 재합류 비용

긴 타임아웃의 위험:
  진짜 장애 시 ttl 만료까지 서비스가 멈춤 → RTO 길어짐

권장 관계 (split-brain 안전):
  ttl  ≥  loop_wait + retry_timeout + (여유)
  예) loop_wait=10, retry_timeout=10  →  ttl=30 (기본값이 이 관계를 만족)
```

- **운영 가이드**: 네트워크가 안정적이고 빠른 복구가 SLA상 중요하면 `ttl`을 20~30s로. 클라우드/공유 인프라처럼 순간 지연(jitter)이 잦으면 오탐 방지를 위해 더 보수적으로. **바꾸기 전 장애 주입 테스트**(노드 kill, 네트워크 지연 주입)로 오탐률을 측정할 것.

### 6.4. etcd Quorum 상실 — DCS가 곧 단일 장애점

**PG 노드는 멀쩡한데 etcd(DCS) 과반수가 깨지면, Patroni는 안전을 위해 Primary를 read-only로 강등한다.** 합의 저장소를 못 믿으면 "내가 진짜 리더인지" 확신할 수 없기 때문(CP 우선).

```
etcd 3노드 중 2노드 장애 → 과반수(2/3) 상실
  → Patroni: leader 키 갱신 불가 → Primary 자진 강등(read-only)
  → 쓰기 전면 중단 (PG는 살아 있는데도!)
  → "데이터 분기보다 쓰기 중단이 낫다"는 의도된 설계

교훈: etcd 클러스터의 HA가 PG HA만큼 중요하다.
  - etcd는 PG 노드와 물리적으로 분리(같이 죽지 않게)
  - 3노드는 1대 장애만 허용, 운영 여유는 5노드(2대 허용)
  - etcd 디스크는 fsync 빠른 SSD (느린 디스크 = 합의 지연 = false failover 유발)
  - etcd 백업(snapshot)도 별도로 — DCS 데이터 유실 = 클러스터 상태 상실
```

### 6.5. 동기 vs 비동기 복제 (RPO 다이얼)

페일오버 시 **데이터를 얼마나 잃을 수 있는가(RPO)** 는 복제 모드가 결정한다.

| 모드 | RPO | 쓰기 지연 | 동작 |
|------|-----|-----------|------|
| **비동기**(기본) | > 0 (수 KB~MB 손실 가능) | 낮음 (빠름) | Primary가 로컬 커밋 후 즉시 ack. Standby 반영은 나중. |
| **동기**(synchronous_commit) | **0** | 높음 (Standby ack 대기) | 최소 1개 Standby가 WAL 받았다고 확인해야 커밋 완료. |

```
비동기 페일오버 시:  Primary가 ack했지만 아직 Standby로 안 넘어간 WAL은
                     승격된 새 Primary에 없다 → 그만큼 데이터 손실(RPO>0).
                     단, maximum_lag_on_failover(§3, 1MB)로 손실 상한을 둔다.

동기 복제:  Patroni synchronous_mode: true
            → 항상 동기 Standby를 1개 유지, 그 노드만 페일오버 후보
            → RPO=0 보장. 단, 동기 Standby가 느리거나 죽으면 쓰기가 막힐 수 있어
              synchronous_mode_strict 여부로 "가용성 vs 무손실" 중 선택.
```

- **SLA 관점 결정**: 금융 원장·결제처럼 한 건도 못 잃으면 **동기**(지연·가용성 비용 감수). 로그·분석·일반 웹 트래픽처럼 약간의 손실이 허용되면 **비동기**(성능·가용성 우선). 같은 클러스터에서 테이블/트랜잭션별로 섞기보다, 클러스터 정책으로 명확히 정하는 편이 운영상 안전.

---

## 7. Spring Boot 연동

```yaml
# application.yml
spring:
  datasource:
    # 쓰기: HAProxy Primary 포트
    url: jdbc:postgresql://haproxy-host:5432/mydb
    username: myapp
    password: changeme

# 읽기 전용 DataSource (선택)
spring:
  datasource-readonly:
    url: jdbc:postgresql://haproxy-host:5433/mydb
    username: myapp
    password: changeme
```

```java
// 읽기/쓰기 DataSource 분리 (선택적 최적화)
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource writeDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://haproxy:5432/mydb")
            .build();
    }

    @Bean("readDataSource")
    public DataSource readDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://haproxy:5433/mydb")
            .build();
    }
}

// 서비스에서 읽기 DS 사용
@Transactional(readOnly = true)
@Service
public class ReportService {
    // readOnly=true 트랜잭션 → 읽기 DataSource 사용 가능
}
```

---

## 8. 모니터링

```bash
# 클러스터 실시간 상태
watch patronictl -c /etc/patroni/patroni.yml list

# etcd 리더 확인
etcdctl --endpoints=... endpoint status --write-out=table

# HAProxy 통계
curl http://haproxy:7000/stats

# Patroni REST API 직접 확인
curl http://pg-node1:8008/patroni    # 전체 클러스터 상태 JSON
curl http://pg-node1:8008/primary    # Primary이면 200
curl http://pg-node1:8008/replica    # Replica이면 200
curl http://pg-node1:8008/health     # 어떤 역할이든 running이면 200
```

---

## 9. 관련
- [[PostgreSQL-Internals]] — VACUUM, 인덱스, EXPLAIN ANALYZE
- [[TimescaleDB]] — Patroni + TimescaleDB 결합 가능
- [[../observability/PLG-Stack]] — PostgreSQL 메트릭 수집 (postgres_exporter)
- [[../k8s/CNPG]] — K8s 환경의 PostgreSQL HA(오퍼레이터 기반 failover) 구현
