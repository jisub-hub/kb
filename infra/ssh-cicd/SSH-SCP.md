---
tags:
  - ssh
  - scp
  - security
  - linux
created: 2026-06-15
---

# SSH & SCP

> [!summary] 한 줄 요약
> **SSH**: 네트워크를 통해 원격 서버에 암호화된 채널로 접속하는 프로토콜. **SCP**: SSH 위에서 동작하는 파일 전송 도구. 클라우드 VM 접속·배포 자동화의 기초.

---

## 1. SSH 개요

- **Secure Shell**: 평문 전송 Telnet의 보안 대안 (1995년 등장)
- 기본 포트: **TCP 22**
- **하이브리드 암호화**: 키 교환(비대칭)으로 세션 키를 협상 → 이후 AES 등 대칭 암호화로 통신
- 기능: 원격 명령 실행, 파일 전송(SCP/SFTP), 포트 포워딩, 터널링

### 연결 수립 순서 (TCP 22 → 암호화 채널)

```
1. 클라이언트 → 서버 TCP 22 연결 요청
2. 서버가 자신의 Host Public Key 전달 (첫 접속 시 신뢰 여부 확인 → ~/.ssh/known_hosts 저장)
3. Diffie-Hellman 키 교환 → 양측이 동일한 대칭 세션 키 생성
4. AES 암호화 채널 수립
5. 클라이언트 인증 (Password 또는 Public Key)
```

> [!warning] known_hosts 경고 무시 금지
> 서버 Host Key가 바뀌면 `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` 경고 발생. 임의로 무시하지 말고 서버 재생성 여부 확인 → 정상이면 `ssh-keygen -R <host>` 로 기존 키 제거 후 재접속.

---

## 2. 접속 방식 비교

### Password 방식
```bash
ssh user@192.168.0.10
# user@192.168.0.10's password: ▌
```

| 항목 | 내용 |
|---|---|
| 장점 | 별도 키 파일 없이 즉시 접속 가능 |
| 단점 | 브루트포스(무작위 대입) 공격에 취약 |
| 단점 | 비밀번호 노출 시 서버 전체 침해 |
| 단점 | Shell Script·CI/CD 자동화 불가 (비밀번호를 코드에 넣어야 함) |
| 클라우드 | OCI·AWS 등 클라우드 VM은 기본적으로 비활성화 |

### PEM Key (공개키) 방식
```bash
ssh -i ~/.ssh/mykey.pem user@192.168.0.10
```

| 항목 | 내용 |
|---|---|
| 장점 | 개인키 없으면 접속 불가 → 안전 |
| 장점 | Shell Script·CI/CD 파이프라인 자동화 가능 |
| 장점 | 클라우드 VM 표준 접속 방식 |
| 주의 | 개인키(`.pem`) 파일 유출 시 서버 침해 → 반드시 권한 600 설정 |
| 주의 | 키 파일 분실 시 재발급(서버 콘솔) 필요 |

```bash
chmod 600 ~/.ssh/mykey.pem    # 필수 — 600 아니면 SSH가 거부함
```

---

## 3. 공개키 / 개인키 등록 절차

```
개인키(Private Key) → 내 컴퓨터에만 보관 (절대 유출 금지)
공개키(Public Key)  → 접속할 서버의 ~/.ssh/authorized_keys 에 등록
```

### STEP 1: 키 쌍 생성

```bash
# RSA 4096비트 (기본 권장)
ssh-keygen -t rsa -b 4096 -C "dev@example.com"

# Ed25519 (더 짧고 현대적, 권장)
ssh-keygen -t ed25519 -C "dev@example.com"

# 생성 위치
~/.ssh/id_rsa      ← 개인키 (절대 공유 금지)
~/.ssh/id_rsa.pub  ← 공개키 (서버에 등록)
```

### STEP 2: 공개키 확인

```bash
cat ~/.ssh/id_rsa.pub
# ssh-rsa AAAAB3NzaC1yc2EA...  dev@example.com
```

### STEP 3: 서버에 공개키 등록

```bash
# 방법 1: ssh-copy-id (권장 — 자동으로 권한까지 설정)
ssh-copy-id -i ~/.ssh/id_rsa.pub user@192.168.0.10

# 방법 2: 수동 등록
cat ~/.ssh/id_rsa.pub | ssh user@host 'cat >> ~/.ssh/authorized_keys'

# 방법 3: 서버에 직접 편집 (이미 접속된 경우)
echo "ssh-rsa AAAA..." >> ~/.ssh/authorized_keys
```

### STEP 4: 서버 권한 설정 (필수)

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
# 권한이 틀리면 sshd가 authorized_keys를 무시함
```

---

## 4. SSH 유용한 옵션

```bash
# 포트 지정 (기본 22 → 변경된 경우)
ssh -p 2222 -i key.pem user@host

# 백그라운드 포트 포워딩 (로컬 8080 → 서버 8080)
ssh -L 8080:localhost:8080 -i key.pem user@host -N

# SSH 터널로 DB 접속 (DB는 직접 노출하지 않고 SSH 통해 접속)
ssh -L 3306:db-server:3306 -i key.pem bastion-user@bastion-host -N
# 이후 로컬에서 localhost:3306 으로 DB 접속

# 점프 호스트(Bastion) 경유 접속
ssh -J bastion-user@bastion-ip target-user@private-ip -i key.pem

# 연결 유지 설정 (~/.ssh/config)
Host my-server
    HostName 192.168.0.10
    User ubuntu
    IdentityFile ~/.ssh/mykey.pem
    ServerAliveInterval 60
```

---

## 5. SCP — 파일 전송

```
scp [옵션] [원본] [대상]
```

SSH와 동일한 포트(22), 동일한 인증 방식 사용.

### 기본 명령어

```bash
# 로컬 → 서버 (업로드)
scp -i key.pem ./app.jar user@192.168.0.10:/opt/app/

# 서버 → 로컬 (다운로드)
scp -i key.pem user@192.168.0.10:/var/log/app.log ./

# 디렉토리 전체 업로드 (-r: recursive)
scp -r -i key.pem ./dist/ user@host:/opt/frontend/

# 디렉토리 전체 다운로드
scp -r -i key.pem user@host:/var/log/ ./logs/

# 다른 포트 지정 (-P 대문자 주의, ssh는 -p 소문자)
scp -P 2222 -i key.pem ./app.jar user@host:/opt/
```

### 주요 옵션

| 옵션 | 설명 |
|---|---|
| `-i <key>` | PEM 키 파일 지정 |
| `-r` | 디렉토리 재귀 전송 |
| `-P <port>` | 포트 지정 (기본 22, **대문자**) |
| `-p` | 파일 권한·타임스탬프 유지 |
| `-v` | 상세 로그 출력 (디버깅) |
| `-C` | 전송 중 압축 (대용량 파일) |

> [!tip] rsync 대안
> 대용량 또는 반복 배포 시 `rsync -avz --delete dist/ user@host:/opt/frontend/` 가 더 효율적. 변경된 파일만 전송하고 삭제된 파일도 동기화.

---

## 6. Shell Script 배포 예시

```bash
#!/bin/bash
# deploy.sh — 빌드 → SCP → 서버 재시작

SERVER="opc@192.168.0.10"
KEY="~/.ssh/id_rsa"
JAR="build/libs/app.jar"
DEPLOY="/opt/myapp"

echo "[1/4] 빌드 중..."
./gradlew clean build -x test

echo "[2/4] 파일 전송..."
scp -i $KEY $JAR $SERVER:$DEPLOY/app-new.jar

echo "[3/4] 서비스 재시작..."
ssh -i $KEY $SERVER "
  systemctl stop myapp &&
  mv $DEPLOY/app-new.jar $DEPLOY/app.jar &&
  systemctl start myapp"

echo "[4/4] 상태 확인..."
ssh -i $KEY $SERVER "systemctl status myapp --no-pager"
```

> [!tip] 배포 전 `app-new.jar` 로 먼저 올린 후 서비스 중지 → 교체 → 시작
> 전송 중 서비스 중단 시간 최소화. 더 나아가면 [[GitLab-CICD]] 로 자동화.

---

## 7. 관련
- [[GitLab-CICD]] · [[../network/Security-ACG-NSG]] · [[../network/HA-Architecture]]
