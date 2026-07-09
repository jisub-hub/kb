---
tags:
  - container
  - docker
  - network
  - volume
  - docker-compose
  - docker-swarm
  - k3s
created: 2026-06-16
---

# Docker 네트워크·볼륨·단일 노드 오케스트레이션

> [!summary] 한 줄 요약
> 컨테이너 네트워크 격리는 보안의 첫 번째 방어선. 볼륨은 컨테이너 재시작 시 데이터를 유지하는 유일한 방법. 단일 노드 프로덕션은 **Docker Compose(소규모) → Docker Swarm(중간) → k3s(K8s 호환 필요)** 순으로 복잡도를 올린다.

---

## 1. Docker 네트워크 격리

### 네트워크 드라이버 비교

| 드라이버 | 용도 | 특징 |
|---------|------|------|
| **bridge** | 단일 호스트 컨테이너 간 통신 | 기본값, 커스텀 브리지 권장 |
| **host** | 호스트 네트워크 그대로 사용 | 포트 매핑 불필요, 보안 약함 |
| **none** | 네트워크 완전 격리 | 외부 통신 불가 (보안 극대화) |
| **overlay** | Swarm 멀티 호스트 통신 | 암호화 옵션, VXLAN |
| **macvlan** | 컨테이너에 MAC 주소 할당 | 물리 네트워크 직접 연결 |

### 기본 bridge vs 커스텀 bridge

```bash
# ❌ 기본 bridge (docker0): 모든 컨테이너가 같은 네트워크
#    컨테이너 이름으로 DNS 해석 안 됨 → IP 직접 지정 필요
docker run --name app1 myapp
docker run --name db1 postgres   # app1에서 db1 이름으로 접근 불가

# ✅ 커스텀 bridge: 이름 기반 DNS, 격리, 서브넷 설정 가능
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.240.0/20 \
  --opt "com.docker.network.bridge.name"="br-app" \
  app-network

docker run --name app1 --network app-network myapp
docker run --name db1  --network app-network postgres
# app1 → db1:5432 으로 직접 접근 가능 (이름 DNS 해석)
```

### 네트워크 격리 전략

```
[격리 레이어 설계 예시]
  frontend-net  ──── nginx, app
  backend-net   ──── app, db, redis
  monitoring-net──── app, prometheus, grafana

  규칙:
    nginx: frontend-net만 (db 직접 접근 불가)
    app:   frontend-net + backend-net (두 네트워크 동시 연결)
    db:    backend-net만 (외부 노출 없음)
```

```yaml
# docker-compose.yml — 네트워크 격리 설계
networks:
  frontend-net:
    driver: bridge
    ipam:
      config: [{ subnet: 172.21.0.0/24 }]

  backend-net:
    driver: bridge
    internal: true      # ← 이 네트워크는 외부 인터넷 접근 완전 차단
    ipam:
      config: [{ subnet: 172.22.0.0/24 }]

services:
  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    networks: [frontend-net]            # frontend-net만 연결

  app:
    image: myapp:latest
    networks: [frontend-net, backend-net]  # 두 네트워크 브리지 역할

  db:
    image: postgres:16-alpine
    networks: [backend-net]             # backend-net만 (internet: false)
    # 5432 포트 외부 노출 없음

  redis:
    image: redis:7-alpine
    networks: [backend-net]
```

### 포트 바인딩 보안

```yaml
# ❌ 위험: 모든 인터페이스에 바인딩 (외부 접근 가능)
ports:
  - "5432:5432"      # 0.0.0.0:5432 → 방화벽 없으면 인터넷 노출!

# ✅ 안전: localhost만 바인딩 (같은 호스트에서만 접근)
ports:
  - "127.0.0.1:5432:5432"

# ✅ 최선: ports 없이 networks만 사용 (컨테이너 간 통신만)
# → 컨테이너 내부 포트만 expose, 호스트에서 직접 접근 불가
expose:
  - "5432"    # 같은 네트워크 컨테이너에서만 접근 가능
```

---

## 2. Docker 볼륨

### 볼륨 종류

```
[Named Volume]                [Bind Mount]             [tmpfs]
  Docker가 관리               호스트 경로 직접           메모리 임시 저장
  /var/lib/docker/volumes/    마운트                    재시작 시 소멸
  → 프로덕션 데이터 저장 권장  → 개발 시 코드 핫리로드    → 비밀 값 임시 처리
                               → 설정 파일 주입           → 캐시

volumes:
  - db-data:/var/lib/postgresql/data    # Named Volume
  - ./config:/etc/nginx/conf.d:ro       # Bind Mount (읽기 전용)
  - type: tmpfs                         # tmpfs
    target: /tmp
```

### Named Volume 생명주기

```bash
# 볼륨 생성
docker volume create --driver local \
  --opt type=none \
  --opt o=bind \
  --opt device=/mnt/ssd/pg-data \   # 특정 디렉토리에 저장
  pg-data-volume

# 볼륨 목록
docker volume ls

# 볼륨 상세 (마운트 경로 확인)
docker volume inspect pg-data-volume

# 미사용 볼륨 정리 (주의: 데이터 삭제)
docker volume prune

# 특정 볼륨 삭제 (컨테이너 중지 후)
docker volume rm pg-data-volume
```

### 볼륨 백업·복원

```bash
# 백업: 볼륨 데이터를 tar로 압축
docker run --rm \
  -v pg-data-volume:/data \         # 백업할 볼륨 마운트
  -v $(pwd)/backups:/backup \       # 백업 파일 저장 경로
  alpine \
  tar czf /backup/pg-data-$(date +%Y%m%d).tar.gz -C /data .

# 복원
docker run --rm \
  -v pg-data-volume:/data \
  -v $(pwd)/backups:/backup \
  alpine \
  tar xzf /backup/pg-data-20260616.tar.gz -C /data

# PostgreSQL pg_dump 방식 (더 안전)
docker exec postgres_container \
  pg_dumpall -U postgres > backup_$(date +%Y%m%d).sql

docker exec -i postgres_container \
  psql -U postgres < backup_20260616.sql
```

### 볼륨 권한 문제 해결

```dockerfile
# Dockerfile — 컨테이너 유저가 볼륨에 쓸 수 있도록
FROM postgres:16-alpine
# postgres 기본 UID=999, GID=999

# VolumMount 시 호스트 디렉토리 권한 맞추기:
# sudo chown -R 999:999 /mnt/ssd/pg-data
```

```yaml
# docker-compose.yml — user 지정으로 권한 일치
services:
  app:
    image: myapp:latest
    user: "1000:1000"         # 호스트 사용자 UID:GID 맞춤
    volumes:
      - ./uploads:/app/uploads  # 호스트 디렉토리 권한이 1000이어야 함
```

---

## 3. Docker Compose — 단일 호스트 운영

### 프로덕션용 Compose 패턴

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  app:
    image: myapp:${APP_VERSION:-latest}    # 버전 태그 주입
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 512M
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s       # 앱 시작 시간 여유
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"         # 로그 파일 최대 5개 로테이션
    environment:
      - SPRING_PROFILES_ACTIVE=prod
    env_file:
      - .env.prod             # 비밀 값 파일 분리
    networks: [backend-net]
    depends_on:
      db:
        condition: service_healthy   # DB 헬스체크 통과 후 시작
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    volumes:
      - pg-data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      retries: 5
    networks: [backend-net]

volumes:
  pg-data:
    driver: local

networks:
  backend-net:
    driver: bridge
    internal: true
```

```bash
# 환경별 설정 합성 (base + override)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 특정 서비스만 재시작 (무중단)
docker compose up -d --no-deps --build app

# 롤링 재시작 (replicas 없을 때 수동)
docker compose up -d --force-recreate app
```

---

## 4. Docker Swarm — 단일·멀티 노드 클러스터

```
특징:
  - Docker 내장 오케스트레이터 (별도 설치 불필요)
  - Compose 파일 그대로 사용 (stack deploy)
  - 단일 노드에서도 배포 전략 사용 가능 (Rolling/Blue-Green)
  - Overlay 네트워크로 멀티 노드 통신
  - Secret 관리 내장 (docker secret)

한계:
  - K8s 대비 생태계·기능 제한
  - Helm, Operator 등 없음
  - Ingress가 단순 (L4 라운드로빈)
```

### Swarm 초기화

```bash
# 단일 노드 Swarm 초기화
docker swarm init

# 멀티 노드: 워커 노드 추가
docker swarm join-token worker    # 토큰 출력
# 워커 노드에서:
docker swarm join --token SWMTKN-... manager-ip:2377

# 노드 목록 확인
docker node ls
```

### Stack 배포 (Compose 파일 재사용)

```bash
# Swarm stack 배포
docker stack deploy -c docker-compose.prod.yml myapp

# 서비스 목록
docker stack services myapp

# 롤링 업데이트
docker service update \
  --image myapp:2.0.0 \
  --update-parallelism 1 \       # 한 번에 1개씩 교체
  --update-delay 10s \           # 교체 간 10초 대기
  --update-failure-action rollback \  # 실패 시 자동 롤백
  myapp_app

# 서비스 스케일 조정
docker service scale myapp_app=3

# 스택 제거
docker stack rm myapp
```

### Swarm Secret (비밀값 관리)

```bash
# Secret 생성
echo "mysupersecretpassword" | docker secret create db_password -

# docker-compose.yml에서 secret 사용
services:
  app:
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password  # 파일로 마운트됨

secrets:
  db_password:
    external: true    # docker secret create로 미리 생성
```

### Swarm Overlay 네트워크 (멀티 노드)

```yaml
networks:
  app-overlay:
    driver: overlay
    encrypted: true       # 노드 간 트래픽 AES 암호화
    attachable: true      # 일반 컨테이너도 연결 가능
```

---

## 5. k3s — 경량 K8s (단일 노드)

```
특징:
  - 정식 K8s의 경량 배포판 (CNCF 인증)
  - 단일 바이너리 ~60MB (K8s etcd → SQLite/Embedded etcd)
  - Raspberry Pi, 엣지 서버, 소형 VM 운영 가능
  - kubectl, Helm 그대로 사용
  - Traefik Ingress 기본 포함
  - K3s → 추후 K8s 클러스터로 마이그레이션 경로 제공

K3s vs K8s:
  동일: Pod, Deployment, Service, ConfigMap, Secret, Ingress, RBAC
  제거: cloud-controller, 일부 볼륨 플러그인, 일부 alpha 기능
  대체: etcd → SQLite (기본), containerd (Docker 불필요)
```

### 설치 및 기본 사용

```bash
# 설치 (단일 명령)
curl -sfL https://get.k3s.io | sh -

# 서비스 상태
systemctl status k3s

# kubeconfig 설정 (일반 사용자)
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
# 또는
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config

# 클러스터 확인
kubectl get nodes
kubectl get pods -A
```

### k3s에 앱 배포

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels: { app: myapp }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # 항상 2개 이상 유지
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: myapp
          image: myapp:1.0.0
          ports: [{ containerPort: 8080 }]
          resources:
            limits:   { cpu: "1", memory: "1Gi" }
            requests: { cpu: "200m", memory: "256Mi" }
          readinessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector: { app: myapp }
  ports: [{ port: 80, targetPort: 8080 }]
---
# k3s 기본 Traefik Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-svc
                port: { number: 80 }
```

### k3s Helm 차트 배포

```bash
# Helm 설치 (k3s 환경)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 예: cert-manager 설치
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# 내 앱 배포
helm install myapp ./charts/myapp \
  -f values.prod.yaml \
  --namespace production --create-namespace
```

---

## 6. 단일 노드 옵션 비교

| 기준 | Docker Compose | Docker Swarm | k3s |
|------|---------------|-------------|-----|
| **복잡도** | 낮음 | 중간 | 높음 |
| **배포 전략** | 수동 | Rolling/Rollback | Rolling/BlueGreen/Canary |
| **서비스 디스커버리** | 컨테이너 이름 DNS | 서비스 이름 DNS | K8s DNS (CoreDNS) |
| **Secret 관리** | .env 파일 | docker secret | K8s Secret/SealedSecret |
| **Ingress** | 수동 Nginx | L4 라운드로빈 | Traefik (L7, TLS) |
| **멀티 노드** | ❌ | ✅ (Overlay) | ✅ (+ K3d 로컬 멀티) |
| **K8s 호환 리소스** | ❌ | ❌ | ✅ 완전 호환 |
| **권장 규모** | 1~5 서비스, 개발 | 5~20 서비스, 소규모 프로덕션 | K8s 필요 + 단일 노드 제약 |

---

## 7. 관련
- [[Container]] — Dockerfile, 이미지 빌드, 보안
- [[Zero-Downtime-Deployment]] — Blue-Green·Canary·Rolling 무중단 배포
- [[../k8s/_index]] — 멀티 노드 K8s 전체 가이드
- [[../linux/Systemd]] — 단일 서버 서비스 관리 (컨테이너 없는 환경)
