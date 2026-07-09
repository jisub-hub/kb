---
tags:
  - deployment
  - blue-green
  - canary
  - rolling-update
  - zero-downtime
  - container
created: 2026-06-16
updated: 2026-06-16
---

# 무중단 배포 원칙 — Docker Compose · Swarm · Nginx · DB 마이그레이션

> [!summary] 한 줄 요약
> K8s 배포 YAML은 [[../k8s/RollingUpdate]] 참고. 이 문서는 **배포 전략의 원칙**, **Docker Compose/Swarm 구현**, **Nginx 트래픽 전환**, **DB 마이그레이션 안전 순서**, **Spring Graceful Shutdown**을 다룬다.

---

## 1. 세 전략 트레이드오프

```
[Rolling Update]
  v1 v1 v1 v1
  → v2 v1 v1 v1   (readiness 통과 후 구버전 1개 제거)
  → v2 v2 v2 v2   완료

[Blue-Green]
  Blue(v1) ← 트래픽 100%
  Green(v2) 준비 완료 → 즉시 전환
  Blue(v1) 대기 (롤백 슬롯)

[Canary]
  stable(v1) ← 90%  +  canary(v2) ← 10%
  → 이상 없으면 점진 확대 → 100%
```

| 기준 | Rolling | Blue-Green | Canary |
|------|---------|-----------|--------|
| **배포 속도** | 점진적 | 즉시 전환 | 점진적 |
| **리소스** | 기존 + α | 기존 × 2 | 기존 + 1~5% |
| **롤백** | 느림 (재배포) | 즉시 | 즉시 (트래픽 0%) |
| **버전 혼용** | 있음 | 없음 | 있음 |
| **DB 마이그레이션** | 주의 | 주의 | 가장 안전 |
| **복잡도** | 낮음 | 중간 | 높음 |

---

## 2. Docker Compose Rolling (수동)

Compose는 자동 롤링이 없다. Nginx 로드밸런서 + 복수 replica 조합으로 수동 구현한다.

```bash
# 전제: nginx + app 서비스, app은 복수 컨테이너

# 1. 새 이미지 빌드
APP_VERSION=2.0.0 docker compose build app

# 2. 인스턴스 수 증가 (기존 유지하며 신버전 추가)
APP_VERSION=2.0.0 docker compose up -d --no-deps --scale app=2 app
sleep 30   # 헬스체크 대기

# 3. 이전 버전 컨테이너 교체
APP_VERSION=2.0.0 docker compose up -d --no-deps --force-recreate app
```

```yaml
# docker-compose.prod.yml
services:
  app:
    image: myapp:${APP_VERSION:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    depends_on:
      app:
        condition: service_healthy
```

---

## 3. Docker Swarm Rolling

```bash
# 서비스 업데이트
docker service update \
  --image myapp:2.0.0 \
  --update-parallelism 1 \           # 1개씩 교체
  --update-delay 15s \               # 교체 간격
  --update-monitor 30s \             # 교체 후 30초 헬스 감시
  --update-failure-action rollback \ # 실패 시 자동 롤백
  --rollback-parallelism 2 \         # 롤백은 2개씩 빠르게
  myapp_app

# 진행 상황 확인
docker service ps myapp_app

# 수동 롤백
docker service rollback myapp_app
```

```yaml
# docker-compose.swarm.yml
version: '3.9'
services:
  app:
    image: myapp:2.0.0
    deploy:
      mode: replicated
      replicas: 4
      update_config:
        parallelism: 1
        delay: 15s
        monitor: 30s
        failure_action: rollback
        order: start-first          # 신버전 먼저 기동 후 구버전 제거 (무중단)
      rollback_config:
        parallelism: 2
        failure_action: pause
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
```

---

## 4. Nginx Blue-Green / Canary (VM · 단일 서버)

### Blue-Green

```nginx
# /etc/nginx/conf.d/myapp.conf
upstream blue  { server 127.0.0.1:8081; }
upstream green { server 127.0.0.1:8082; }

server {
    listen 80;
    server_name myapp.example.com;

    # ← 이 변수만 바꾸고 nginx -s reload
    set $active_upstream blue;

    location / {
        proxy_pass http://$active_upstream;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout 2s;
        proxy_read_timeout 60s;
    }
}
```

```bash
# Green(8082)에 신버전 기동 후 트래픽 전환
sed -i 's/\$active_upstream blue/\$active_upstream green/' \
    /etc/nginx/conf.d/myapp.conf
nginx -t && nginx -s reload

# 롤백
sed -i 's/\$active_upstream green/\$active_upstream blue/' \
    /etc/nginx/conf.d/myapp.conf
nginx -s reload
```

### Canary (가중치)

```nginx
upstream myapp {
    server 127.0.0.1:8081 weight=9;   # stable v1: 90%
    server 127.0.0.1:8082 weight=1;   # canary v2: 10%
    keepalive 16;
}

server {
    listen 80;
    location / {
        proxy_pass http://myapp;
    }
}
```

```bash
# 비율 50:50으로 변경
# upstream 블록 weight 수정 후
nginx -s reload
```

---

## 5. DB 마이그레이션 안전 순서

배포 중 v1 + v2 코드가 동시에 실행될 수 있다(Rolling·Canary). **비호환 스키마 변경은 3 사이클에 나눠** 진행해야 한다.

```
[위험한 패턴]
  v2 코드: SELECT id, email FROM users (email 컬럼 신규)
  v1 코드: SELECT id, name FROM users  (email 모름)

  → v2 배포 중 v1 인스턴스가 email-required INSERT 실패

[안전한 3-Cycle 패턴: 컬럼 이름 변경 예시]
  name → full_name 으로 변경한다고 가정

  1 Cycle (호환 추가)
    마이그레이션: full_name 컬럼 추가 (nullable)
    v2 코드: name + full_name 양쪽 다 쓰기, 읽기는 COALESCE(full_name, name)
    → v1(name만 씀) + v2(양쪽) 동시 운영 가능

  2 Cycle (기존 제거 준비)
    v3 코드: full_name만 사용, name 컬럼 쓰기 중단
    → 모든 인스턴스가 v3로 전환 완료 확인

  3 Cycle (구 컬럼 삭제)
    마이그레이션: name 컬럼 DROP
    → v3 코드만 실행 중이므로 안전

[Blue-Green에서 DB 처리]
  전환 순간 v1 → v2 완전 교체이므로 중간 상태 없음
  단, 롤백 시 DB 롤백 불가 → 모든 마이그레이션은 하위호환 필수
  (롤백 가능성 있으면 컬럼 삭제를 별도 사이클로 분리)
```

---

## 6. 배포 전략 선택 기준

```
[서비스 성숙도 / 팀 역량]
  초기 스타트업, 트래픽 소량:
    → Rolling Update (가장 단순, 관리 오버헤드 최소)

  서비스 안정화, DB 변경 잦음:
    → Blue-Green (즉시 롤백, DB 상태 명확)

  대규모 트래픽, 중요 변경:
    → Canary + Argo Rollouts (자동 메트릭 게이트)

[DB 마이그레이션 여부]
  없음:            Rolling (가장 단순)
  하위 호환 변경:   Rolling 또는 Canary
  비호환 변경:      Blue-Green (전환 순간 스키마 일치)

[인프라 환경]
  단일 VM/베어메탈: Nginx Blue-Green / Canary
  Docker Swarm:     service update (내장 롤링)
  Kubernetes:       → [[../k8s/RollingUpdate]] 참고
```

---

## 7. Spring Boot Graceful Shutdown

배포 교체 시 구 인스턴스가 **처리 중인 요청을 완료하고** 종료해야 한다.

```yaml
# application.yml
server:
  shutdown: graceful             # immediate(기본) → graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # 30초 대기 후 강제 종료
```

```java
// 비동기 작업·외부 연결을 명시적으로 정리해야 하는 경우
@Component
@Slf4j
public class GracefulShutdownHandler {

    @PreDestroy
    public void onShutdown() {
        log.info("Graceful shutdown: finishing in-flight work...");
        // 진행 중인 비동기 태스크 완료 대기
        // MQ consumer stop, DB connection drain
    }
}
```

> **K8s 설정 맞추기**: `preStop: exec: sleep 10` + `terminationGracePeriodSeconds: 60`이 `timeout-per-shutdown-phase`(30s)보다 커야 한다.

---

## 8. 관련
- [[../k8s/RollingUpdate]] — K8s Deployment YAML, Blue-Green Service selector, Argo Rollouts 설정
- [[Docker-Network-Volume]] — Docker Swarm, k3s, 네트워크 격리
- [[../observability/PLG-Stack]] — 배포 후 에러율·지연시간 모니터링
