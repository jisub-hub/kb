---
tags:
  - deploy
  - docker
  - container
created: 2026-06-15
---

# Docker

> [!summary] 한 줄 요약
> 애플리케이션과 의존성을 **이미지**로 패키징해 어디서나 동일하게 실행하는 컨테이너 플랫폼. "내 PC에선 됐는데" 문제를 제거하고 [[Kubernetes]] 배포의 단위가 된다.

---

## 1. 핵심 개념
- **Image**: 실행에 필요한 파일시스템 스냅샷(불변, 레이어 구조).
- **Container**: 이미지의 실행 인스턴스.
- **Registry**: 이미지 저장소(Docker Hub, ECR, GHCR).
- **Dockerfile**: 이미지 빌드 명세.

## 2. Spring Boot용 멀티스테이지 Dockerfile
```dockerfile
# ── build stage ──
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY gradlew settings.gradle build.gradle ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon || true   # 의존성 레이어 캐시
COPY src ./src
RUN ./gradlew bootJar --no-daemon

# ── runtime stage (작은 이미지) ──
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
> 멀티스테이지로 JDK(빌드)와 JRE(실행)를 분리 → 최종 이미지 경량화.

### 더 쉬운 방법 — Buildpacks (Dockerfile 불필요)
```bash
./gradlew bootBuildImage --imageName=myorg/order-service:1.0
# Spring Boot가 최적화된 OCI 이미지를 자동 생성 (레이어 분리 포함)
```

## 3. 주요 명령
```bash
docker build -t myorg/order:1.0 .
docker run -d -p 8080:8080 --name order \
  -e SPRING_PROFILES_ACTIVE=prod myorg/order:1.0
docker logs -f order
docker exec -it order sh
docker push myorg/order:1.0
```

## 4. docker-compose (로컬 통합 환경)
```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
    depends_on: [db, redis]
  db:
    image: postgres:16
    environment: { POSTGRES_DB: mydb, POSTGRES_PASSWORD: secret }
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: valkey/valkey:8           # [[Valkey]] / [[Redis]]
volumes: { pgdata: {} }
```

## 5. 베스트 프랙티스
- **작은 베이스 이미지**(jre, distroless, alpine 주의: glibc).
- **레이어 캐시 활용**(의존성 → 소스 순서).
- **non-root 유저**로 실행, 시크릿은 이미지에 넣지 말 것(env/secret).
- `.dockerignore` 로 불필요 파일 제외.
- 컨테이너는 **stateless** + 로그는 **stdout**([[Logging]]).
- 헬스체크: Spring Actuator `/actuator/health` 연동.

## 6. 관련
- [[Kubernetes]] · [[CICD]] · [[Logging]] · [[MSA]]
