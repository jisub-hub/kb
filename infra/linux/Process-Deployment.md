---
tags:
  - linux
  - deployment
  - nohup
  - systemd
  - container
  - process-management
created: 2026-06-16
---

# 프로세스 배포 방식 — nohup · systemd · Container

> [!summary] 한 줄 요약
> **nohup**은 가장 단순한 백그라운드 실행 (임시·개발), **systemd**는 OS 레벨 서비스 관리 (서버 재부팅 자동 시작·로그 통합), **Container**는 환경 표준화·확장성 (프로덕션 표준). 복잡도·운영 편의성·이식성 순으로 선택한다.

---

## 1. 세 방식 비교

| 기준 | nohup | systemd | Container (Docker) |
|------|-------|---------|-------------------|
| **난이도** | 매우 쉬움 | 중간 | 중간~높음 |
| **서버 재부팅 시** | 수동 재시작 필요 | ✅ 자동 시작 | ✅ restart policy |
| **로그 관리** | 파일 직접 관리 | ✅ journald 통합 | ✅ Docker log driver |
| **환경 격리** | ❌ 없음 | 제한적 | ✅ 완전 격리 |
| **버전 롤백** | 수동 | 수동 | ✅ 이미지 태그 |
| **의존성 관리** | ❌ 없음 | 서비스 의존성 | ✅ 이미지 포함 |
| **리소스 제한** | ❌ 없음 | cgroups 설정 가능 | ✅ --memory, --cpus |
| **모니터링** | ps/top | systemctl status | docker stats |
| **적합한 환경** | 빠른 테스트 | VM 서비스 운영 | 표준 프로덕션 |

---

## 2. nohup

SSH 세션이 끊겨도 프로세스가 계속 실행되도록 하는 가장 단순한 방법.

```bash
# 기본 사용
nohup java -jar myapp.jar &

# stdout, stderr 모두 파일로
nohup java -jar myapp.jar > /var/log/myapp.log 2>&1 &

# 실행 중인 PID 확인
echo $!          # 직전에 실행한 백그라운드 PID
# 또는
ps aux | grep myapp.jar

# 종료
kill -15 <PID>   # SIGTERM (graceful)
kill -9 <PID>    # SIGKILL (강제)
```

### PID 파일로 관리

```bash
#!/bin/bash
# start.sh
PID_FILE="/var/run/myapp.pid"
LOG_FILE="/var/log/myapp.log"

if [ -f "$PID_FILE" ] && kill -0 $(cat $PID_FILE) 2>/dev/null; then
    echo "이미 실행 중 (PID: $(cat $PID_FILE))"
    exit 1
fi

nohup java -Xms512m -Xmx1g \
    -jar /opt/myapp/myapp.jar \
    --spring.profiles.active=prod \
    > "$LOG_FILE" 2>&1 &

echo $! > "$PID_FILE"
echo "기동 완료 (PID: $!)"

# stop.sh
if [ -f "$PID_FILE" ]; then
    kill -15 $(cat "$PID_FILE")
    rm -f "$PID_FILE"
    echo "종료 완료"
fi
```

### nohup 단점

```
❌ 서버 재부팅 시 자동 시작 안 됨 (crontab @reboot으로 부분 해결 가능)
❌ 로그 로테이션 직접 설정 (logrotate)
❌ 환경변수·설정 관리 어려움
❌ 의존성 (DB, MQ) 순서 보장 없음
→ 개발 환경, 임시 테스트, 단순 스크립트 용도
```

---

## 3. systemd

Linux의 표준 프로세스 관리자. **Unit 파일**로 서비스를 선언적으로 정의한다.

### Unit 파일 작성

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Spring Boot Application
After=network.target postgresql.service   # 의존성: 네트워크·DB 기동 후 시작

[Service]
Type=simple
User=appuser                              # 전용 사용자로 실행 (root 금지)
Group=appuser
WorkingDirectory=/opt/myapp

# 환경변수 (파일로 분리 권장)
EnvironmentFile=/etc/myapp/env.conf
Environment=JAVA_OPTS=-Xms512m -Xmx1g

ExecStart=/usr/bin/java \
    $JAVA_OPTS \
    -jar /opt/myapp/myapp.jar \
    --spring.profiles.active=prod

# Graceful Shutdown (SIGTERM → 30초 대기 → SIGKILL)
KillSignal=SIGTERM
TimeoutStopSec=30
KillMode=mixed

# 재시작 정책
Restart=on-failure          # 비정상 종료 시 재시작
RestartSec=5s               # 5초 후 재시작
StartLimitBurst=3           # 최대 3회 재시도
StartLimitIntervalSec=60s   # 60초 내 3회 초과 시 중단

# 리소스 제한
MemoryLimit=2G
CPUQuota=200%               # 2코어

# stdout → journald
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target  # 멀티유저 모드에서 활성화
```

```ini
# /etc/myapp/env.conf (환경변수 파일)
DB_URL=jdbc:postgresql://localhost:5432/mydb
DB_PASSWORD=secret
SPRING_PROFILES_ACTIVE=prod
```

### systemd 관리 명령

```bash
# 서비스 등록 및 시작
systemctl daemon-reload                 # unit 파일 변경 후 반드시
systemctl enable myapp                  # 부팅 시 자동 시작 등록
systemctl start myapp                   # 즉시 시작
systemctl enable --now myapp            # 등록 + 즉시 시작 동시

# 상태 확인
systemctl status myapp
# 출력: Active(running)/Failed/Activating, PID, 메모리, 최근 로그

# 로그 확인
journalctl -u myapp                     # 전체 로그
journalctl -u myapp -f                  # 실시간
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp -n 100              # 최근 100줄

# 재시작 / 중단
systemctl restart myapp
systemctl stop myapp
systemctl disable myapp                 # 부팅 시 시작 해제

# 의존성 트리 확인
systemctl list-dependencies myapp
```

### 여러 인스턴스 (Template Unit)

```ini
# /etc/systemd/system/worker@.service
[Unit]
Description=Worker Instance %i
After=network.target

[Service]
ExecStart=/opt/worker/worker.sh %i      # %i = 인스턴스 이름

[Install]
WantedBy=multi-user.target
```

```bash
# 인스턴스별 시작
systemctl start worker@1
systemctl start worker@2
systemctl start worker@gpu
```

---

## 4. Container (Docker)

컨테이너는 앱 + 런타임 + 의존성을 이미지로 패키징. 어디서든 동일하게 실행된다.

### 기본 실행

```bash
# 기본 실행
docker run -d \
  --name myapp \
  --restart=unless-stopped \         # 재부팅 시 자동 시작
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_URL=jdbc:postgresql://db:5432/mydb \
  -v /var/log/myapp:/app/logs \       # 로그 볼륨 마운트
  --memory=1g \
  --cpus=1.0 \
  myorg/myapp:1.2.0

# 상태 확인
docker ps
docker logs myapp -f
docker stats myapp
```

### Docker Compose (멀티 서비스)

```yaml
# docker-compose.yml
services:
  app:
    image: myorg/myapp:1.2.0
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_URL: jdbc:postgresql://db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: mydb
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

volumes:
  db_data:
```

```bash
# Compose 관리
docker compose up -d              # 백그라운드 실행
docker compose down               # 중단 + 컨테이너 제거
docker compose ps                 # 상태 확인
docker compose logs app -f        # 실시간 로그
docker compose restart app        # 재시작

# 이미지 업데이트 (무중단 단일 서버)
docker compose pull app
docker compose up -d --no-deps app
```

### Restart Policy 비교

```
no:              재시작 안 함 (기본값)
always:          항상 재시작 (정상 종료·재부팅 포함)
unless-stopped:  수동으로 중지하지 않으면 항상 재시작 ← 권장
on-failure:      비정상 종료(exit code ≠ 0) 시만 재시작
on-failure:3     최대 3회만
```

---

## 5. 실전 선택 기준

```
[nohup 선택]
  → 빠른 테스트, 개발 서버 임시 기동
  → systemd, Docker 설치 없는 환경
  → 수명이 짧은 스크립트 실행

[systemd 선택]
  → 컨테이너 없이 VM에서 직접 서비스 운영
  → OS 부팅과 긴밀하게 통합 (DB 서버, 에이전트 등)
  → 여러 서비스 의존성 순서 보장 필요
  → journald 기반 중앙화 로그 관리

[Container 선택]
  → 프로덕션 표준 (환경 재현성 보장)
  → 개발/스테이징/운영 환경 일치 필요
  → 멀티 서비스 오케스트레이션 (Compose, K8s)
  → CI/CD 파이프라인과 연계

[조합 패턴]
  Container + systemd:
    docker.service는 systemd가 관리
    개별 컨테이너는 docker run + --restart=always
    또는 docker-compose@.service template unit으로 Compose 자체를 systemd 서비스로
```

### docker-compose를 systemd 서비스로

```ini
# /etc/systemd/system/myapp-compose.service
[Unit]
Description=MyApp Docker Compose
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

---

## 6. 관련
- [[Systemd]] — systemd unit 파일 상세, timer, journalctl
- [[../container/Container]] — Docker 이미지 빌드, Dockerfile
- [[../container/Docker-Network-Volume]] — 컨테이너 네트워크·볼륨 관리
- [[../k8s/_index]] — 컨테이너 오케스트레이션 (프로덕션 확장)
