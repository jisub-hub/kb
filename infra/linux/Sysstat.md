---
tags:
  - linux
  - monitoring
  - performance
  - sysstat
  - sar
created: 2026-06-16
---

# sysstat — Linux 성능 모니터링

> [!summary] 한 줄 요약
> **sysstat**는 `sar`, `iostat`, `mpstat`, `pidstat` 등을 포함하는 Linux 성능 통계 패키지. 과거 이력 저장 기능이 핵심 — 장애 발생 당시 CPU·메모리·디스크·네트워크 상태를 사후 분석 가능.

---

## 1. 설치 및 활성화

```bash
# RHEL/CentOS/Rocky
dnf install sysstat -y
systemctl enable --now sysstat

# Ubuntu/Debian
apt install sysstat -y
# /etc/default/sysstat 에서 ENABLED="true" 로 변경
sed -i 's/ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
systemctl enable --now sysstat

# 수집 주기 설정 (/etc/cron.d/sysstat)
# 기본: 10분마다 수집, 하루 단위 파일 저장
# /var/log/sa/ 에 sa01~sa31 형태로 저장
```

---

## 2. sar — System Activity Reporter (핵심)

### 2.1 CPU 사용률

```bash
# 현재부터 3초 간격 5회 수집
sar -u 3 5

# 출력:
# 10:00:01  CPU   %user  %nice  %system  %iowait  %steal  %idle
# 10:00:04  all    15.2    0.0      3.1      1.2     0.0    80.5

# 항목 설명:
#   %user    — 사용자 공간 프로세스 CPU 사용
#   %system  — 커널 공간 (시스템 콜) CPU 사용
#   %iowait  — I/O 대기 (디스크 병목 지표)
#   %steal   — 가상화 환경에서 하이퍼바이저에게 뺏긴 CPU
#   %idle    — 유휴 (낮을수록 부하 높음)

# CPU별 상세 (-P ALL)
sar -u -P ALL 1 3
# 0:30:01  CPU  %user  %system  %iowait  %idle
# 0:30:02    0   25.0      5.0      0.0   70.0
# 0:30:02    1   10.0      2.0      0.0   88.0

# 과거 데이터 조회 (어제 CPU 사용률)
sar -u -f /var/log/sa/sa$(date -d yesterday +%d)

# 오늘 전체 이력
sar -u
```

### 2.2 메모리

```bash
sar -r 3 5

# 출력:
# kbmemfree  kbavail  kbmemused  %memused  kbbuffers  kbcached  kbcommit  %commit
#   512000  2048000    6144000      92.3     102400   2048000   7168000     98.5

# 항목 설명:
#   kbmemfree   — 완전히 비어있는 메모리 (작아도 정상, Linux는 캐시 활용)
#   kbavail     — 실제 사용 가능 메모리 (free + 재사용 가능 캐시)  ← 핵심 지표
#   %memused    — 전체 메모리 사용률
#   kbcached    — 파일 시스템 캐시 (필요시 반환 가능)
#   %commit     — 100% 초과 시 OOM(Out of Memory) 위험

# 스왑 사용량
sar -S 3 5
# kbswpfree  kbswpused  %swpused  kbswpcad
# → %swpused 증가 = 메모리 부족, 성능 급락 신호
```

### 2.3 디스크 I/O

```bash
sar -d 3 5       # 장치별 I/O
sar -b 3 5       # 전체 I/O 요약

# sar -d 출력:
# DEV    tps    rkB/s    wkB/s  areq-sz  aqu-sz  await  svctm  %util
# sda   150.0  4096.0  2048.0    40.96    2.50   15.0    5.0   75.0

# 항목 설명:
#   tps      — 초당 I/O 요청 수 (Transactions Per Second)
#   rkB/s    — 초당 읽기 KB
#   wkB/s    — 초당 쓰기 KB
#   areq-sz  — 평균 요청 크기 (KB)
#   aqu-sz   — 평균 요청 대기열 크기 (높을수록 디스크 포화)
#   await    — 평균 I/O 대기 시간 (ms) ← 핵심. SSD: <1ms, HDD: <20ms
#   %util    — 디스크 활용률 (%100에 가까울수록 포화)

# 핵심 판단:
#   await > 20ms (HDD) or > 5ms (SSD) + %util > 80% → 디스크 병목
```

### 2.4 네트워크

```bash
sar -n DEV 3 5        # 네트워크 인터페이스별
sar -n EDEV 3 5       # 네트워크 에러
sar -n SOCK 3 5       # 소켓 통계
sar -n TCP 3 5        # TCP 연결 통계

# sar -n DEV 출력:
# IFACE  rxpck/s  txpck/s  rxkB/s  txkB/s  rxcmp/s  txcmp/s  rxmcst/s  %ifutil
# eth0   1000.0   800.0    4096.0  2048.0       0.0      0.0       0.0    40.96

# 항목:
#   rxpck/s  — 초당 수신 패킷
#   txkB/s   — 초당 송신 KB
#   %ifutil  — 인터페이스 대역폭 활용률

# sar -n EDEV — 에러 지표:
# rxerr/s  txerr/s  coll/s  rxdrop/s  txdrop/s
# → rxdrop/s, txdrop/s 증가 = NIC 버퍼 초과 또는 패킷 유실
```

### 2.5 로드 평균 및 프로세스

```bash
sar -q 3 5

# 출력:
# runq-sz  plist-sz  ldavg-1  ldavg-5  ldavg-15  blocked
#       5      1234     2.50     1.80      1.20        0

# 항목:
#   runq-sz   — 실행 대기 중인 프로세스 수
#   ldavg-1   — 1분 평균 로드 (CPU 코어 수와 비교)
#   blocked   — I/O 대기로 블록된 프로세스 수 (높으면 디스크 병목)

# ldavg 해석: CPU 16코어 서버에서
#   ldavg-1 = 8  → 50% 부하 (여유)
#   ldavg-1 = 16 → 100% 부하 (포화)
#   ldavg-1 = 32 → 200% 부하 (과부하)
```

---

## 3. iostat — 디스크 I/O 상세

```bash
# 기본: 전체 디바이스 요약
iostat -x 3 5

# 출력 (extended):
# Device  r/s    w/s   rkB/s   wkB/s  rrqm/s  wrqm/s  %rrqm  %wrqm  r_await  w_await  aqu-sz  rareq-sz  wareq-sz  svctm  %util
# nvme0n1 500.0  300.0 20480.0 12288.0   0.0    10.0    0.0    3.2     0.5      1.2     0.40     40.96     40.96    0.6  48.0

# 핵심 지표:
#   r_await / w_await — 읽기/쓰기 평균 대기 시간 (ms)
#   aqu-sz            — 평균 큐 깊이 (>1이면 포화 시작)
#   %util             — 장치 활용률

# 단위 변경 (MB/s)
iostat -x -m 3 5

# 특정 디바이스만
iostat -x nvme0n1 3 5

# 장치 없이 CPU 포함
iostat -c -d 3 5
```

---

## 4. mpstat — CPU 코어별 상세

```bash
# 모든 CPU 코어별 1초 간격 5회
mpstat -P ALL 1 5

# 출력:
# CPU   %usr  %nice  %sys  %iowait  %irq  %soft  %steal  %guest  %idle
#   0   25.0    0.0   5.0      2.0   0.5    0.5     0.0     0.0   67.0
#   1   10.0    0.0   2.0      0.5   0.0    0.0     0.0     0.0   87.5

# 핵심 확인:
#   %soft (software interrupt) 높음 → 네트워크 패킷 처리 과부하
#   %irq  (hardware interrupt) 높음 → 디바이스 인터럽트 과다
#   특정 코어만 100% → 싱글스레드 병목
```

---

## 5. pidstat — 프로세스별 상세

```bash
# 모든 프로세스 CPU 사용률 (2초 간격 3회)
pidstat 2 3

# 특정 프로세스 (PID 또는 이름)
pidstat -p 1234 1 5
pidstat -C java 1 5        # 프로세스 이름 필터

# I/O 사용량 (-d)
pidstat -d 1 5
# PID  kB_rd/s  kB_wr/s  kB_ccwr/s  Command
# 1234   512.0   256.0       10.0    java

# 메모리 (-r)
pidstat -r 1 5
# PID  minflt/s  majflt/s  VSZ(kB)  RSS(kB)  %MEM  Command
# 1234    100.0       0.0  2048000  512000   6.25  java

# 스레드별 (-t)
pidstat -t -p 1234 1 3

# 컨텍스트 스위치 (-w)
pidstat -w 1 5
# PID  cswch/s  nvcswch/s  Command
# 1234    50.0      200.0  java
# cswch   — 자발적 컨텍스트 스위치 (I/O 대기 등)
# nvcswch — 비자발적 (CPU 선점)
```

---

## 6. 실전 장애 분석 패턴

### 6.1 CPU 병목

```bash
# 1단계: 전체 CPU 확인
sar -u 1 10
# %idle < 20%, %system > 30% → 시스템 콜 과다 확인

# 2단계: 코어별 불균형 확인
mpstat -P ALL 1 5
# 특정 코어만 100% → 싱글스레드 프로세스 병목

# 3단계: 원인 프로세스 특정
pidstat -u 1 10 | sort -k8 -rn | head -10
top -b -n1 | head -20

# 4단계: 해당 프로세스 스택 확인
jstack <pid>  (Java)
strace -p <pid> -c  (시스템 콜 통계)
perf top  (커널 레벨 핫스팟)
```

### 6.2 메모리 부족 / OOM

```bash
# 1단계: 메모리 사용량 추이
sar -r 1 10
# kbavail 감소 추세, %commit > 100% → OOM 임박

# 2단계: 스왑 확인
sar -S 1 10
# kbswpused 증가 → 메모리 부족, I/O 지연 동반

# 3단계: 프로세스별 메모리
pidstat -r 1 10 | sort -k8 -rn | head -10

# 4단계: OOM 발생 여부
dmesg | grep -i "oom\|killed"
grep -i "oom\|out of memory" /var/log/messages

# 5단계: Java GC 과다 확인
pidstat -u -p <java_pid> 1 10  # CPU 급등
jstat -gcutil <pid> 1000 10    # GC 빈도·시간
```

### 6.3 디스크 I/O 병목

```bash
# 1단계: I/O wait 확인
sar -u 1 10 | awk '{print $6}'   # %iowait 컬럼

# 2단계: 디바이스별 상태
iostat -x 1 10
# await > 20ms (HDD) or > 2ms (SSD) + %util > 80% → 병목

# 3단계: I/O 유발 프로세스
pidstat -d 1 10 | sort -k5 -rn | head -10
iotop -b -n3  (실시간)

# 4단계: 파일시스템 확인
df -h          # 디스크 꽉 참 여부
lsof | wc -l   # 열린 파일 수
```

### 6.4 네트워크 포화

```bash
# 1단계: 인터페이스 처리량
sar -n DEV 1 10
# txkB/s, rxkB/s 로 대역폭 사용률 계산

# 2단계: 패킷 드롭 확인
sar -n EDEV 1 10
# rxdrop/s, txdrop/s 증가 → 버퍼 초과

# 3단계: TCP 연결 상태
sar -n TCP 1 10
ss -s          # 연결 요약
ss -nt state established | wc -l  # 활성 연결 수

# 4단계: 소켓 상태 분포
ss -nt | awk '{print $1}' | sort | uniq -c
# TIME_WAIT 과다 → Keep-alive 설정 또는 소켓 재사용 설정 검토
```

---

## 7. 과거 데이터 분석 — 장애 사후 분석

```bash
# 특정 날짜 데이터 파일
ls /var/log/sa/
# sa01 sa02 ... sa31  (날짜별)
# sar01 sar02 ... (일부 배포판은 sar 접두사)

# 어제 전체 CPU 이력
sar -u -f /var/log/sa/sa$(date -d yesterday +%d)

# 특정 날짜 (5일 전)
sar -u -f /var/log/sa/sa$(date -d '5 days ago' +%d)

# 특정 시간 범위
sar -u -s 09:00:00 -e 12:00:00 -f /var/log/sa/sa15

# 모든 지표 한 번에
sar -A -f /var/log/sa/sa15 > /tmp/full_report.txt

# 사람이 읽기 쉬운 포맷으로 출력
sadf -d /var/log/sa/sa15 -- -u | head -20    # CSV
sadf -j /var/log/sa/sa15 -- -u               # JSON
```

---

## 8. 한눈에 보기 — 지표별 임계치 가이드

| 지표 | 명령 | 경고 | 위험 |
|---|---|---|---|
| CPU %idle | `sar -u` | < 30% | < 10% |
| CPU %iowait | `sar -u` | > 10% | > 30% |
| 메모리 kbavail | `sar -r` | < 10% 전체 | < 5% |
| Swap %swpused | `sar -S` | > 10% | > 50% |
| Disk await | `iostat -x` | > 10ms(SSD) / > 20ms(HDD) | > 50ms |
| Disk %util | `iostat -x` | > 70% | > 90% |
| Disk aqu-sz | `iostat -x` | > 1 | > 4 |
| NIC rxdrop/s | `sar -n EDEV` | > 0 | 지속 증가 |
| Load avg / 코어수 | `sar -q` | > 0.7 | > 1.0 |
| TCP 소켓 TIME_WAIT | `ss -nt` | > 10,000 | > 50,000 |

---

## 9. 관련
- [[Kernel-Parameters]] · [[Ulimit]] · [[../../programming/spring/observability/Metrics]]
