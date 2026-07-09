---
tags:
  - linux
  - ulimit
  - tuning
  - performance
created: 2026-06-15
---

# ulimit — 프로세스 자원 제한

> [!summary] 한 줄 요약
> **ulimit**은 프로세스(또는 사용자)가 사용할 수 있는 OS 자원의 **상한선**. 기본값은 보수적으로 잡혀있어, 고성능 서버(WAS, DB, Redis, Nginx)를 운영하면 이 한계에 부딪히는 경우가 많다.

---

## 1. ulimit이란

- Linux 커널이 각 프로세스/사용자에게 부여하는 **자원 사용 한도**
- 남용 방지 (한 프로세스가 시스템 자원 독점 차단) + **보안 경계** 역할
- 기본값이 작아서 **프로덕션 서버에서는 반드시 조정 필요**

### Soft Limit vs Hard Limit

```
Soft Limit: 현재 적용되는 실질적 한계. 프로세스/사용자가 자신의 Soft를 Hard 이하로 올릴 수 있음
Hard Limit: Soft의 상한선. root만 Hard를 올릴 수 있음

일반 사용자: Soft ≤ Hard 범위 내에서 자유롭게 조정
root:        Hard를 커널 한계까지 올릴 수 있음
```

```bash
ulimit -a          # 현재 셸의 모든 제한 확인
ulimit -Hn         # Hard limit for nofile
ulimit -Sn         # Soft limit for nofile
```

---

## 2. 주요 항목과 왜 건드는가

### `nofile` — 열 수 있는 파일 디스크립터 수

```bash
ulimit -n          # 현재 확인 (기본 1024)
```

**왜 건드는가:**
- Linux에서 **소켓·파일·파이프 모두 파일 디스크립터(fd)** 를 소비
- Spring Boot WAS: 각 HTTP 연결 = fd 1개. DB 커넥션 풀 = fd 수십 개
- Nginx: 각 연결 = fd 2개 (클라이언트 + 업스트림)
- `1024`는 동시 연결 수백 개도 금방 소진 → `Too many open files` 에러

**변경 시 달라지는 것:**
```
nofile: 1024  → 동시 커넥션 ~500개 한계 (소켓 외 fd도 사용하므로)
nofile: 65536 → 동시 커넥션 수만 개 처리 가능
nofile: 1048576 → 고성능 프록시/게이트웨이 수준
```

**설정 예시:**
```bash
# /etc/security/limits.conf
*    soft    nofile    65536
*    hard    nofile    65536

# 특정 사용자만
ubuntu    soft    nofile    1048576
ubuntu    hard    nofile    1048576
```

---

### `nproc` — 생성할 수 있는 최대 프로세스/스레드 수

```bash
ulimit -u          # 현재 확인 (기본 ~1024~4096)
```

**왜 건드는가:**
- Java 앱은 스레드를 많이 사용 (각 스레드 = 프로세스로 카운트)
- Tomcat 스레드 풀 200 + Spring Batch 쓰레드 + GC 스레드 = 수백 개 스레드
- 기본 `nproc=1024`이면 **`OutOfMemoryError: unable to create new native thread`**
- Spring Boot + 여러 작업 스레드가 있는 앱에서 자주 발생

**변경 시 달라지는 것:**
```
nproc: 1024  → 스레드 수백 개에서 신규 스레드 생성 실패
nproc: 65536 → 일반 WAS 충분
nproc: unlimited → 제한 없음 (fork bomb 위험이 있어 root에만 허용 권장)
```

---

### `stack` — 스레드 스택 크기

```bash
ulimit -s          # 현재 확인 (기본 8192KB = 8MB)
```

**왜 건드는가:**
- 각 스레드마다 스택 메모리 예약
- `8MB × 200 스레드 = 1.6GB` → JVM의 Thread Stack 총량에 영향
- 재귀 호출이 깊은 앱: StackOverflowError
- 반대로 스레드를 많이 쓰는데 스택이 크면 메모리 낭비

**변경:**
```bash
# JVM에서는 -Xss로 스레드 스택 직접 지정 (ulimit -s보다 우선)
-Xss512k    # 스레드당 512KB (기본 8MB에서 줄임 → 메모리 절감)
```

---

### `core` — core dump 파일 크기

```bash
ulimit -c          # 기본 0 (core dump 비활성화)
```

**왜 건드는가:**
- `0`이면 프로세스 크래시 시 core dump 파일 생성 안 됨 → 원인 분석 불가
- 프로덕션에서는 `0`으로 두거나, 분석 환경에서는 `unlimited` 설정

```bash
ulimit -c unlimited    # 개발/스테이징에서 크래시 분석용
```

---

### `as` (address space) / `memlock` — 가상 메모리 / 잠금 메모리

**`memlock`:**
```bash
ulimit -l          # 기본 64KB
```
- Elasticsearch, Redis, 일부 DB: 성능을 위해 메모리를 **물리 메모리에 잠금(mlock)** 필요
- `memlock=unlimited` 없으면 mlockall() 실패 → 페이지 스왑 발생 → 성능 저하

```bash
# Elasticsearch, Redis 권장
*    soft    memlock    unlimited
*    hard    memlock    unlimited
```

---

## 3. 설정 방법

### 방법 1: `/etc/security/limits.conf` (영구, PAM 기반 로그인)

```
# /etc/security/limits.conf
# <domain>  <type>  <item>   <value>
*           soft    nofile   65536
*           hard    nofile   65536
*           soft    nproc    65536
*           hard    nproc    65536

# 특정 사용자
ubuntu      soft    nofile   1048576
ubuntu      hard    nofile   1048576

# root는 별도 (도메인 * 는 root에 적용 안 됨)
root        soft    nofile   65536
root        hard    nofile   65536
```

> [!warning] `*`는 root에 적용되지 않는다
> root 사용자는 별도로 명시해야 한다.

### 방법 2: `/etc/security/limits.d/*.conf` (별도 파일로 관리 — 권장)

```bash
# /etc/security/limits.d/99-app.conf
ubuntu  soft  nofile  1048576
ubuntu  hard  nofile  1048576
ubuntu  soft  nproc   65536
ubuntu  hard  nproc   65536
```

### 방법 3: systemd 서비스 파일 (데몬에 직접 적용 — 가장 확실)

```ini
# /etc/systemd/system/myapp.service
[Service]
LimitNOFILE=1048576
LimitNPROC=65536
LimitMEMLOCK=infinity

# 적용
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

> [!tip] systemd 서비스는 PAM 무관
> systemd로 실행되는 데몬(Nginx, Docker, Java 서비스)은 `/etc/security/limits.conf`가 **적용되지 않는다**. 반드시 `systemd` Unit 파일의 `[Service]` 섹션에 직접 설정해야 한다.

### 방법 4: 현재 셸 세션에서 즉시 변경 (임시)

```bash
ulimit -n 65536       # 현재 셸과 이 셸에서 실행한 프로세스에만 적용
ulimit -n unlimited   # 무제한 (Hard limit 범위 내)
```

---

## 4. 커널 전체 fd 한계 (`fs.file-max`)

ulimit은 프로세스/사용자 단위 제한. 시스템 전체 fd 한계는 별도.

```bash
# 현재 시스템 전체 fd 한계 확인
cat /proc/sys/fs/file-max
# 예: 9223372036854775807 (거의 무제한)

# 조정 (sysctl)
sysctl -w fs.file-max=2097152
echo "fs.file-max=2097152" >> /etc/sysctl.conf
```

---

## 5. 설정 후 확인

```bash
# 현재 프로세스의 실제 적용 제한 확인
cat /proc/<pid>/limits

# 예시
Limit                     Soft Limit           Hard Limit
Max open files            1048576              1048576
Max processes             65536                65536
Max locked memory         unlimited            unlimited

# 현재 열린 fd 수 확인
ls /proc/<pid>/fd | wc -l

# 시스템 전체 현재 사용 중인 fd 수
cat /proc/sys/fs/file-nr
# 열린fd  사용중  최대
# 4352    0       9223372036854775807
```

---

## 6. 서비스별 권장 설정 요약

| 서비스 | nofile | nproc | memlock | 비고 |
|---|---|---|---|---|
| **Nginx** | 65536+ | 기본 | - | worker_connections 값 × 2 이상 |
| **Spring Boot (Tomcat)** | 65536 | 65536 | - | 스레드 풀 × 안전 마진 |
| **Redis** | 65536 | 65536 | unlimited | mlock 성능 향상 |
| **Elasticsearch** | 65536 | 4096 | unlimited | ES 공식 권장 |
| **PostgreSQL** | 65536 | 기본 | - | max_connections 고려 |
| **Kafka** | 128000 | 65536 | - | 파티션 수만큼 fd 사용 |

---

## 7. 관련
- [[Kernel-Parameters]] · [[JVM-Tuning]]
