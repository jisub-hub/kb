---
tags:
  - linux
  - sysctl
  - kernel
  - tuning
  - network
created: 2026-06-15
---

# 커널 파라미터 튜닝 (sysctl)

> [!summary] 한 줄 요약
> **sysctl**은 운영 중인 Linux 커널의 파라미터를 동적으로 조회·변경하는 도구. 네트워크 연결 처리, 파일 디스크립터, 메모리 관리 등 OS 수준의 성능 병목을 해결할 때 사용한다.

---

## 1. sysctl 기본 사용법

```bash
# 특정 파라미터 조회
sysctl net.core.somaxconn

# 모든 파라미터 조회
sysctl -a

# 즉시 변경 (재부팅 후 사라짐)
sysctl -w net.core.somaxconn=65535

# 영구 적용
echo "net.core.somaxconn=65535" >> /etc/sysctl.conf
sysctl -p              # /etc/sysctl.conf 즉시 적용

# 권장: 별도 파일로 분리 관리
echo "net.core.somaxconn=65535" > /etc/sysctl.d/99-app.conf
sysctl --system        # /etc/sysctl.d/*.conf 모두 적용
```

---

## 2. 네트워크 파라미터

### 연결 큐 크기

```bash
# TCP SYN 큐 (3-way handshake 대기 연결 수)
net.ipv4.tcp_max_syn_backlog = 65536
# 기본: 512~1024. 트래픽 폭증 시 SYN 패킷 드롭 → 접속 실패
# 왜 건드는가: Nginx/WAS 앞단에서 갑작스러운 트래픽 급증 시 드롭 방지

# 완전히 수립된 연결 대기 큐 (accept 전)
net.core.somaxconn = 65535
# 기본: 128. Nginx worker_connections · Tomcat acceptCount와 맞춰야 함
# 왜 건드는가: 큐 초과 시 클라이언트에 RST 반환 → 연결 거부

# 소켓 수신/송신 버퍼 기본/최대 크기
net.core.rmem_default = 262144
net.core.rmem_max     = 16777216
net.core.wmem_default = 262144
net.core.wmem_max     = 16777216
# 왜 건드는가: 대용량 파일 전송, 고대역폭 환경에서 처리량 향상
```

### TCP 연결 상태 및 재사용

```bash
# TIME_WAIT 소켓을 새 연결에 재사용 허용
net.ipv4.tcp_tw_reuse = 1
# 왜 건드는가: 단시간 대량 HTTP 연결 시 TIME_WAIT 소켓 고갈 방지
# TIME_WAIT는 2MSL(기본 60초) 동안 포트를 점유 → 포트 고갈 가능

# 최대 TIME_WAIT 버킷 수
net.ipv4.tcp_max_tw_buckets = 65536
# 초과 시 강제 소멸 (로그: "TCP: time wait bucket table overflow")

# Outbound 포트 범위 (클라이언트 포트 풀)
net.ipv4.ip_local_port_range = 1024 65535
# 기본: 32768 60999 → 약 28,000개 포트만 사용 가능
# 왜 건드는가: 외부 서비스 대량 호출 시 포트 고갈
# 변경 후 가용 포트: 64,512개로 증가

# FIN_WAIT2 상태 타임아웃 (기본 60초)
net.ipv4.tcp_fin_timeout = 15
# 왜 건드는가: 연결 종료 지연 소켓을 빠르게 정리 → 자원 회수
```

### TCP Keepalive

```bash
# Keepalive 활성화 후 유휴 감지까지 대기 시간
net.ipv4.tcp_keepalive_time    = 600   # 기본 7200초(2시간) → 10분으로 단축
net.ipv4.tcp_keepalive_intvl   = 30    # 재시도 간격 (기본 75초)
net.ipv4.tcp_keepalive_probes  = 3     # 재시도 횟수 (기본 9)

# 왜 건드는가:
# - 죽은 연결(half-open) 조기 감지 → DB/외부 API 커넥션 풀 건강 유지
# - 기본값(2시간)은 너무 길어 방화벽이 먼저 끊어버림
# - keepalive_time=600이면 10분 유휴 후 감지 시작 → 3×30초 = 90초 내 판정
```

### 백로그 및 패킷 처리

```bash
# 네트워크 카드 수신 큐 크기
net.core.netdev_max_backlog = 65536
# 왜 건드는가: NIC → 커널 처리 속도 차이로 패킷 드롭 방지. 고트래픽 서버

# 동시 연결 추적 테이블 크기 (Conntrack, NAT 사용 시)
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 300
# 왜 건드는가: Docker/k8s 환경에서 iptables/NAT 쓰면 conntrack 테이블 고갈 가능
# "nf_conntrack: table full, dropping packet" 에러 시 설정
```

---

## 3. 파일시스템 파라미터

```bash
# 시스템 전체 최대 파일 디스크립터 수
fs.file-max = 2097152
# 기본: 수백만 이상이라 보통 건드릴 일 없음
# ulimit -n 이 먼저 걸림

# inotify 감시 수 (파일 변경 감지)
fs.inotify.max_user_watches = 524288
# 기본: 8192. IntelliJ, Webpack HMR, ArgoCD, Prometheus 등이 많은 파일 감시
# "inotify watch limit reached" 에러 시 설정. 개발 서버에서 자주 발생

fs.inotify.max_user_instances = 512
fs.inotify.max_queued_events  = 32768
```

---

## 4. 가상 메모리 파라미터

```bash
# 스왑 사용 적극성 (0=스왑 최소, 100=적극 사용)
vm.swappiness = 10
# 기본: 60. 서버에서 스왑 사용은 성능 급락
# 왜 건드는가: DB, Redis, JVM 등 메모리 의존 앱에서 스왑 방지
# 0으로 설정하면 OOM 발생 시 스왑 없이 바로 OOM killer

# Dirty 페이지(수정됐지만 아직 디스크에 안 쓴) 비율 상한
vm.dirty_ratio           = 15   # 기본 20
vm.dirty_background_ratio = 5   # 기본 10
# 왜 건드는가:
# dirty_ratio 초과 시 쓰기 작업 블로킹 → 애플리케이션 멈춤(쓰기 대기)
# DB 서버: 낮게 설정 → 주기적으로 빨리 flush → 갑작스러운 블로킹 방지

# 메모리 오버커밋 정책
vm.overcommit_memory = 1
# 0: 커널이 적당히 판단 (기본)
# 1: 항상 허용 (Redis, fork 많이 쓰는 앱 권장)
# 2: 물리 메모리 + 스왑 이상 할당 불가
# 왜 건드는가: Redis는 RDB 저장 시 fork() 사용 → overcommit=0이면 메모리 부족 에러

# 투명 거대 페이지 (THP) — Redis, MongoDB 권장: 비활성화
# THP는 큰 페이지 사용으로 메모리 효율 높임, 하지만 Redis/MongoDB에서 지연 유발
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# 영구 적용: /etc/rc.local 또는 systemd 서비스로 처리
```

---

## 5. 전체 적용 예시 — 고성능 WAS 서버

```bash
# /etc/sysctl.d/99-was-tuning.conf

# 연결 큐
net.core.somaxconn               = 65535
net.ipv4.tcp_max_syn_backlog     = 65536
net.core.netdev_max_backlog      = 65536

# 버퍼
net.core.rmem_max                = 16777216
net.core.wmem_max                = 16777216
net.ipv4.tcp_rmem                = 4096 87380 16777216
net.ipv4.tcp_wmem                = 4096 65536 16777216

# 연결 재사용 및 정리
net.ipv4.tcp_tw_reuse            = 1
net.ipv4.tcp_fin_timeout         = 15
net.ipv4.ip_local_port_range     = 1024 65535
net.ipv4.tcp_max_tw_buckets      = 65536

# Keepalive
net.ipv4.tcp_keepalive_time      = 600
net.ipv4.tcp_keepalive_intvl     = 30
net.ipv4.tcp_keepalive_probes    = 3

# 파일시스템
fs.file-max                      = 2097152
fs.inotify.max_user_watches      = 524288

# 메모리
vm.swappiness                    = 10
vm.dirty_ratio                   = 15
vm.dirty_background_ratio        = 5
```

```bash
sysctl --system   # 적용
```

---

## 6. 설정 전 현재 상태 확인

```bash
# TCP 연결 상태별 카운트
ss -s
# 또는
netstat -an | awk '/^tcp/ {print $6}' | sort | uniq -c | sort -rn

# 현재 TIME_WAIT 수
ss -tan | grep TIME-WAIT | wc -l

# 포트 고갈 여부 (에러 카운터)
netstat -s | grep -i "failed\|overflow\|drop"

# 파일 디스크립터 사용 현황
cat /proc/sys/fs/file-nr
# [열린 수] [사용 중] [최대]
```

---

## 7. 관련
- [[Ulimit]] · [[Ramdisk]] · [[../network/Network-Basics]]
