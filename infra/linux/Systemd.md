---
tags:
  - linux
  - systemd
  - service
  - journal
created: 2026-06-16
---

# Systemd — 서비스 관리 & 로그

> [!summary] 한 줄 요약
> systemd는 Linux의 **PID 1 init 시스템**. 서비스 시작/정지/재시작, 의존성 관리, 소켓 활성화, 타이머, 그리고 **journald**로 구조화 로그를 제공한다.

---

## 1. systemctl 핵심 명령어

```bash
# 서비스 관리
systemctl start   myapp.service    # 시작
systemctl stop    myapp.service    # 정지
systemctl restart myapp.service    # 재시작 (종료 후 시작)
systemctl reload  myapp.service    # 설정 리로드 (프로세스 유지)
systemctl status  myapp.service    # 상태 + 최근 로그

# 부팅 시 자동 시작
systemctl enable  myapp.service    # 심볼릭 링크 생성
systemctl disable myapp.service    # 링크 제거
systemctl is-enabled myapp.service

# 시스템 상태
systemctl list-units --type=service --state=running  # 실행 중인 서비스
systemctl list-units --failed                         # 실패한 서비스
systemctl daemon-reload              # unit 파일 변경 후 반드시 실행

# 타겟(구 런레벨)
systemctl get-default                # 현재 기본 타겟
systemctl set-default multi-user.target   # GUI 없는 서버 모드
systemctl isolate rescue.target      # 복구 모드
```

---

## 2. Unit 파일 작성

```bash
# unit 파일 위치
/etc/systemd/system/myapp.service    # 관리자 정의 (최우선)
/usr/lib/systemd/system/             # 패키지 설치 기본값
```

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Spring Boot Application
Documentation=https://docs.example.com
After=network.target postgresql.service   # postgresql 기동 후 시작
Wants=postgresql.service                  # 권장 의존성 (없어도 시작)
Requires=network.target                   # 필수 의존성 (없으면 실패)

[Service]
Type=simple                   # forking | oneshot | notify | idle
User=appuser                  # 실행 사용자 (root 금지)
Group=appuser
WorkingDirectory=/opt/myapp

# 환경변수
EnvironmentFile=/etc/myapp/env          # 파일에서 로드 (비밀값 분리)
Environment="JAVA_OPTS=-Xmx2g -Xms512m"

ExecStart=/usr/bin/java $JAVA_OPTS -jar /opt/myapp/app.jar
ExecStop=/bin/kill -TERM $MAINPID       # 기본값 SIGTERM
ExecReload=/bin/kill -HUP $MAINPID

# 재시작 정책
Restart=on-failure            # 비정상 종료 시 자동 재시작
RestartSec=5s                 # 재시작 전 대기
StartLimitIntervalSec=60s     # 60초 내
StartLimitBurst=3             # 3번 이상 실패 시 포기

# 리소스 제한
LimitNOFILE=65536             # ulimit -n (open files)
LimitNPROC=4096               # ulimit -u (processes)
MemoryMax=4G                  # cgroup 메모리 상한
CPUQuota=200%                 # CPU 2코어 상당

# 보안 강화
ProtectSystem=strict          # /usr, /boot, /etc 읽기 전용
ProtectHome=true              # /home, /root 접근 차단
NoNewPrivileges=true          # 권한 상승 차단
PrivateTmp=true               # 전용 /tmp 사용
ReadWritePaths=/var/lib/myapp /var/log/myapp  # 쓰기 허용 경로 명시

# 표준출력 → journald
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp        # journalctl -t myapp 으로 필터 가능

[Install]
WantedBy=multi-user.target
```

```bash
# 적용
systemctl daemon-reload
systemctl enable --now myapp.service    # enable + start 동시
systemctl status myapp.service
```

---

## 3. EnvironmentFile — 비밀값 분리

```bash
# /etc/myapp/env  (권한 600, appuser 소유)
DB_URL=jdbc:postgresql://localhost:5432/mydb
DB_PASSWORD=supersecret
JWT_SECRET=abcdef1234567890
SPRING_PROFILES_ACTIVE=production
```

```bash
chmod 600 /etc/myapp/env
chown appuser:appuser /etc/myapp/env
```

---

## 4. Systemd Timer (cron 대체)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily DB Backup Timer

[Timer]
OnCalendar=daily                     # 매일 자정
OnCalendar=*-*-* 02:00:00            # 매일 02:00
OnBootSec=5min                       # 부팅 5분 후 1회
RandomizedDelaySec=30min             # 지터 (분산 실행)
Persistent=true                      # 놓친 실행 기동 후 실행

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily DB Backup

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup.sh
```

```bash
systemctl enable --now backup.timer
systemctl list-timers                # 타이머 목록 + 다음 실행 시각
```

---

## 5. journalctl — 로그 조회

```bash
# 기본 조회
journalctl -u myapp.service          # 특정 서비스 로그
journalctl -u myapp.service -f       # 실시간 팔로우
journalctl -u myapp.service -n 100   # 마지막 100줄
journalctl -u myapp.service --since "2026-06-16 09:00" --until "2026-06-16 10:00"

# 레벨 필터
journalctl -u myapp.service -p err   # 에러 이상만 (emerg|alert|crit|err|warning|notice|info|debug)

# 부팅별 로그
journalctl -b          # 현재 부팅 로그
journalctl -b -1       # 직전 부팅 로그
journalctl --list-boots

# Syslog identifier
journalctl -t myapp    # SyslogIdentifier=myapp 기준 필터

# 특정 PID / 유닛 / 프로세스
journalctl _PID=1234
journalctl _SYSTEMD_UNIT=myapp.service _PRIORITY=3   # 에러 레벨

# JSON 출력 (로그 파이프라인 연동)
journalctl -u myapp.service -o json-pretty | jq '.MESSAGE'

# 디스크 사용량 & 정리
journalctl --disk-usage
journalctl --vacuum-size=500M    # 500MB 초과분 삭제
journalctl --vacuum-time=2weeks  # 2주 이상 로그 삭제
```

---

## 6. 서비스 의존성 시각화

```bash
systemctl list-dependencies myapp.service           # 의존 트리
systemctl list-dependencies myapp.service --reverse # 역방향 (이 서비스에 의존하는 것)
systemd-analyze                                      # 부팅 시간 분석
systemd-analyze blame                                # 서비스별 부팅 기여 시간
systemd-analyze critical-chain                       # 부팅 크리티컬 패스
```

---

## 7. 관련
- [[Ulimit]] · [[Kernel-Parameters]] · [[Sysstat]] · [[LVM-Storage]]
