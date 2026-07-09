---
tags:
  - linux
  - ramdisk
  - tmpfs
  - performance
  - storage
created: 2026-06-15
---

# Ramdisk — tmpfs & ramfs

> [!summary] 한 줄 요약
> **Ramdisk**: 물리 메모리(RAM)를 디스크처럼 마운트해 사용하는 가상 파일시스템. 디스크 I/O가 완전히 없어 **읽기/쓰기 속도가 RAM 속도(수십 GB/s)**에 근접한다. 단, **전원이 꺼지면 내용이 사라진다(휘발성)**.

---

## 1. tmpfs vs ramfs

Linux에서 Ramdisk를 구현하는 두 가지 방식.

| | **tmpfs** | **ramfs** |
|---|---|---|
| 동작 방식 | 가상 파일시스템 (VFS), 페이지 캐시 사용 | 초기 RAM 기반 파일시스템 |
| 크기 제한 | ✅ 있음 (마운트 시 지정 가능) | ❌ 없음 (메모리 꽉 찰 때까지 증가) |
| 스왑 사용 | ✅ 메모리 부족 시 스왑 가능 | ❌ 절대 스왑 안 됨 |
| 용량 확인 | `df -h`로 가능 | 어려움 |
| 권장 여부 | ✅ **실무에서 tmpfs 사용** | ⚠️ 크기 제한 없어 위험 |

> [!danger] ramfs는 크기 제한이 없어 메모리를 전부 소진할 수 있다
> 프로덕션에서는 항상 **tmpfs**를 사용. ramfs는 임베디드 시스템 초기 부팅 등 특수 목적에만.

---

## 2. tmpfs 마운트

### 임시 마운트 (재부팅 시 사라짐)

```bash
# /mnt/ramdisk 에 2GB tmpfs 마운트
sudo mkdir -p /mnt/ramdisk
sudo mount -t tmpfs -o size=2g tmpfs /mnt/ramdisk

# 확인
df -h /mnt/ramdisk
# tmpfs  2.0G  0  2.0G  0%  /mnt/ramdisk

# 마운트 해제
sudo umount /mnt/ramdisk
```

### 영구 마운트 (`/etc/fstab`)

```bash
# /etc/fstab 에 추가
tmpfs   /mnt/ramdisk   tmpfs   defaults,size=2g,uid=1000,gid=1000,mode=0755   0  0

# 바로 마운트 (fstab 적용)
sudo mount /mnt/ramdisk
```

### 마운트 옵션 상세

| 옵션 | 설명 |
|---|---|
| `size=2g` | 최대 크기 2GB (초과 불가) |
| `size=50%` | 물리 RAM의 50% |
| `uid=1000` | 마운트 포인트 소유자 UID |
| `gid=1000` | 마운트 포인트 소유자 GID |
| `mode=0755` | 디렉토리 권한 |
| `noexec` | 실행 파일 실행 금지 (보안) |
| `nosuid` | setuid 비트 무시 (보안) |

---

## 3. 성능 비교

| 스토리지 | 순차 읽기 | 순차 쓰기 | 랜덤 4K IOPS |
|---|---|---|---|
| HDD (7200rpm) | ~150 MB/s | ~150 MB/s | ~100 |
| SATA SSD | ~550 MB/s | ~520 MB/s | ~90,000 |
| NVMe SSD | ~7,000 MB/s | ~6,500 MB/s | ~1,000,000 |
| **tmpfs (RAM)** | **~50,000 MB/s** | **~50,000 MB/s** | **사실상 무제한** |

→ tmpfs는 NVMe SSD보다도 **10배 이상 빠르다**. 디스크 I/O가 병목인 작업에서 극적인 효과.

---

## 4. 활용 사례

### 빌드 캐시 / 임시 파일

```bash
# Maven/Gradle 빌드 임시 디렉토리를 ramdisk에
sudo mount -t tmpfs -o size=4g tmpfs /tmp/build-cache

# Gradle 예시
./gradlew build -Djava.io.tmpdir=/tmp/build-cache
```

CI 빌드 서버에서 빌드 속도 향상에 효과적.

---

### Nginx / 로그 임시 저장

```bash
# Nginx temp 디렉토리를 tmpfs에
# /etc/fstab
tmpfs   /var/cache/nginx   tmpfs   defaults,size=512m,uid=www-data,gid=www-data   0  0
```

```nginx
# nginx.conf
proxy_temp_path   /var/cache/nginx/proxy_temp;
client_body_temp_path /var/cache/nginx/client_temp;
```

대용량 파일 업로드 처리 시 디스크 대신 RAM 버퍼 사용 → I/O 부하 감소.

---

### 테스트용 DB 데이터 디렉토리

```bash
# 통합 테스트용 PostgreSQL 데이터를 tmpfs에
sudo mount -t tmpfs -o size=1g tmpfs /var/lib/postgresql/test

# 또는 Docker
docker run -d \
  --tmpfs /var/lib/postgresql/data:rw,size=1g \
  -e POSTGRES_PASSWORD=test \
  postgres:16
```

디스크 I/O 없이 **테스트 실행 속도 대폭 향상**. 테스트 후 자동 초기화 (tmpfs).

---

### `/tmp` 를 tmpfs로 (많은 배포판 기본)

```bash
# Ubuntu 22.04+는 기본적으로 /tmp가 tmpfs
df -h /tmp
# tmpfs  1.9G  1.2M  1.9G  1%  /tmp

# 비활성화된 경우 활성화
sudo systemctl enable tmp.mount
sudo systemctl start tmp.mount
```

---

### JVM 임시 파일 디렉토리

```bash
# JVM이 사용하는 임시 디렉토리를 tmpfs로
JAVA_OPTS="-Djava.io.tmpdir=/mnt/ramdisk"
```

---

## 5. 주의사항

### 휘발성 (Volatile) — 가장 중요

```
⚠️ 재부팅 또는 umount 시 모든 데이터 즉시 소멸
```

- 영구 보존이 필요한 데이터는 절대 tmpfs에만 저장하면 안 됨
- 캐시·임시 파일·빌드 산출물·테스트 데이터 등 **재생성 가능한 것만** 저장
- 주기적으로 영구 스토리지에 동기화 필요 시 `rsync` cron 설정

### 메모리 압박

```bash
# tmpfs가 실제 사용 중인 메모리 확인
df -h /mnt/ramdisk          # 할당된 크기
free -h                     # 전체 메모리 현황

# 페이지 캐시 포함 실제 메모리 사용량
cat /proc/meminfo | grep -E "MemTotal|MemFree|Cached|SwapUsed"
```

- tmpfs 크기를 물리 RAM의 **50% 이하**로 유지 권장
- 남은 RAM이 부족해지면 tmpfs 내용이 스왑으로 밀릴 수 있음

### 크기 초과 시

```bash
# tmpfs 마운트 크기 초과하면 일반 디스크 full처럼
# "No space left on device" 에러 발생

# 런타임 크기 변경 (데이터 유지하며 조정 가능)
sudo mount -o remount,size=4g /mnt/ramdisk
```

---

## 6. 컨테이너 환경 (Docker/k8s)

### Docker

```bash
# 컨테이너에 tmpfs 마운트
docker run -d \
  --tmpfs /tmp:rw,size=512m,mode=1777 \
  myapp:latest
```

### Kubernetes

```yaml
spec:
  volumes:
    - name: ramdisk
      emptyDir:
        medium: Memory      # tmpfs 사용
        sizeLimit: 512Mi
  containers:
    - name: myapp
      volumeMounts:
        - name: ramdisk
          mountPath: /tmp/cache
```

> `medium: Memory`를 지정하면 k8s가 자동으로 tmpfs로 마운트. 미지정 시 일반 디스크(`emptyDir`).

---

## 7. 관련
- [[Ulimit]] · [[Kernel-Parameters]] · [[../k8s/PV-PVC]]
