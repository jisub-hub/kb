---
tags:
  - infra
  - container
  - docker
created: 2026-06-15
---

# Container & Docker

> [!summary] 한 줄 요약
> 애플리케이션 + 의존성을 **이미지**로 패키징해 어디서나 동일하게 실행하는 기술. OS를 공유하므로 VM보다 가볍고 빠르다.

---

## 1. 핵심 개념

```
[Registry]  ←push / pull→  [Image]  →run→  [Container]
                               ↑
                          Dockerfile 로 빌드
```

| 용어 | 설명 |
|------|------|
| **Image** | 파일시스템 레이어 스택 (불변) |
| **Container** | 이미지의 실행 인스턴스 (읽기+쓰기 레이어 추가) |
| **Registry** | 이미지 저장소 (Docker Hub, ECR, GHCR) |
| **Layer** | Dockerfile 명령어 한 줄 = 레이어 하나. 캐시 단위 |
| **Daemon** | 백그라운드 서버 (`dockerd`). 모든 docker 명령을 처리 |

---

## 2. Dockerfile 주요 명령어

```dockerfile
FROM eclipse-temurin:21-jre          # 베이스 이미지 (마지막 FROM이 최종)
WORKDIR /app                          # 이후 명령의 작업 디렉토리
COPY src/ ./src/                      # 호스트 → 이미지
ADD app.tar.gz /app/                  # COPY + 압축 해제·URL 지원 (COPY 우선 권장)
RUN apt-get update && apt-get install -y curl  # 레이어 생성. &&로 묶어 레이어 최소화
ENV APP_ENV=production                # 환경변수 (런타임에도 유효)
EXPOSE 8080                           # 문서용 포트 선언 (실제 열리진 않음)
ENTRYPOINT ["java", "-jar", "app.jar"] # 컨테이너 기본 실행 명령 (항상 실행)
CMD ["--spring.profiles.active=prod"] # ENTRYPOINT 인자 / 단독으로도 사용 가능
```

### 멀티스테이지 빌드 — 이미지 경량화
```dockerfile
# 빌드 스테이지 (JDK 포함)
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

# 실행 스테이지 (JRE만)
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 3. 주요 CLI 명령어

```bash
# 이미지
docker build -t myapp:1.0 .              # 현재 디렉토리 Dockerfile로 빌드
docker images                            # 로컬 이미지 목록
docker rmi myapp:1.0                     # 이미지 삭제
docker pull nginx:alpine                 # 레지스트리에서 풀
docker push myorg/myapp:1.0              # 레지스트리에 푸시

# 컨테이너
docker run -d \                          # -d: 백그라운드
  -p 8080:8080 \                         # 호스트:컨테이너 포트 매핑
  -e SPRING_PROFILES_ACTIVE=prod \       # 환경변수
  -v /host/data:/app/data \              # 볼륨 마운트
  --name myapp \
  myapp:1.0

docker ps                                # 실행 중인 컨테이너
docker ps -a                             # 전체 컨테이너
docker logs -f myapp                     # 로그 스트리밍
docker exec -it myapp bash               # 컨테이너 내부 접속
docker stop myapp && docker rm myapp     # 정지 + 삭제
```

---

## 4. Docker Compose — 멀티 컨테이너 로컬 환경

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_REDIS_HOST: redis
    depends_on:
      db: { condition: service_healthy }
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_PASSWORD: secret
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  redis:
    image: valkey/valkey:8-alpine

volumes:
  pgdata:
```

```bash
docker compose up -d        # 백그라운드 기동
docker compose logs -f app  # 특정 서비스 로그
docker compose down -v      # 컨테이너 + 볼륨 삭제
```

---

## 5. 네트워킹과 볼륨

**네트워크**
| 드라이버 | 설명 | 사용처 |
|---------|------|--------|
| `bridge` | 기본. 가상 네트워크 격리 | 로컬 단일 호스트 |
| `host` | 호스트 네트워크 공유 | 성능 우선, Linux only |
| `overlay` | 멀티 호스트 간 네트워크 | Docker Swarm / Kubernetes |

**볼륨**
```bash
docker volume create mydata
docker run -v mydata:/app/data myapp   # named volume (Docker 관리)
docker run -v /host/path:/app/data     # bind mount (호스트 경로 직접)
```

---

## 6. 컨테이너 보안

### 보안 강화 Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre AS base

# 전용 비-루트 유저 생성
RUN groupadd -r appgroup && useradd -r -g appgroup -s /sbin/nologin appuser

WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar

# 앱이 쓰는 디렉토리만 명시적으로 권한 부여
RUN mkdir -p /app/logs /tmp/app && \
    chown -R appuser:appgroup /app /tmp/app

# non-root로 전환
USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

### .dockerignore — 빌드 컨텍스트 최소화

```
# .dockerignore
.git/
.github/
target/
build/
*.md
*.log
.env
.env.*
**/*.key
**/*.pem
secrets/
node_modules/
```

### Distroless vs Alpine 이미지

| | 일반 JRE | Alpine JRE | Distroless |
|--|---------|-----------|------------|
| 크기 | ~250MB | ~100MB | ~80MB |
| 쉘 | 있음 | `/bin/sh` | **없음** |
| 패키지 관리자 | apt | apk | **없음** |
| 보안 | 기본 | 작은 표면 | **최소 공격 표면** ⭐ |
| 디버깅 | 쉬움 | 보통 | 어려움 |

```dockerfile
# Distroless — 프로덕션 보안 강화
FROM gcr.io/distroless/java21-debian12:nonroot
COPY --from=build /app/build/libs/*.jar /app.jar
ENTRYPOINT ["/app.jar"]
```

### 이미지 취약점 스캔

```bash
# Trivy 스캔 (CI/CD 통합)
trivy image --exit-code 1 --severity CRITICAL myapp:1.2.3

# Docker Scout (Docker Hub 연동)
docker scout cves myapp:1.2.3
docker scout recommendations myapp:1.2.3   # 베이스 이미지 업그레이드 제안
```

### docker run 보안 옵션

```bash
docker run \
  --read-only \                            # 루트 파일시스템 읽기 전용
  --tmpfs /tmp:size=100m \                 # 임시 디렉토리만 쓰기 허용
  --security-opt no-new-privileges \       # 권한 상승 차단
  --cap-drop ALL \                         # 모든 Linux capabilities 제거
  --cap-add NET_BIND_SERVICE \             # 필요한 것만 추가
  --user 1000:1000 \                       # non-root UID:GID
  --memory 512m --cpus 0.5 \              # 자원 제한
  myapp:1.2.3
```

---

## 7. 베스트 프랙티스

- **레이어 캐시 순서**: 변경 빈도 낮은 것(의존성) → 높은 것(소스코드) 순으로 COPY
- **non-root 유저**: `USER appuser` 로 권한 최소화 — 루트 컨테이너는 탈출 시 호스트 루트 획득 위험
- **시크릿 금지**: 이미지에 패스워드·키 포함 ❌ → 런타임 env/K8s Secret으로
- **.dockerignore**: `.git/`, `target/`, `.env` 제외 — 소스코드·비밀값 이미지 내 포함 방지
- **컨테이너는 Stateless**: 상태는 외부 볼륨·DB에 위임, 로그는 stdout
- **버전 태그 고정**: `latest` 금지 → `eclipse-temurin:21-jre-jammy` 형태로 명시
- **이미지 서명**: Cosign/Sigstore → [[../../programming/security/Supply-Chain-Security]]

---

## 관련
- [[Kubernetes]] · [[Security-Context]] · [[RBAC]]
- [[../../programming/security/Supply-Chain-Security]] · [[infra/_index]]
