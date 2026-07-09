---
tags:
  - cctv
  - mediamtx
  - rtsp
  - rtmp
  - hls
  - webrtc
  - streaming-server
created: 2026-06-16
---

# MediaMTX — 범용 미디어 서버

> [!summary] 한 줄 요약
> **MediaMTX** (구 rtsp-simple-server)는 단일 바이너리로 **RTSP, RTMP, HLS, WebRTC, SRT** 다섯 프로토콜을 동시에 처리하는 오픈소스 미디어 서버. IP 카메라 → RTSP 수신 → HLS/WebRTC 재배포 패턴으로 CCTV 웹 모니터링 시스템 구축 핵심.

---

## 1. MediaMTX 개요

```
이전 이름: rtsp-simple-server (2023년 MediaMTX로 변경)
언어: Go (단일 바이너리, 의존성 없음)
라이선스: MIT

지원 프로토콜:
  - RTSP 1.0 (포트 8554)
  - RTMP / RTMPS (포트 1935)
  - HLS (포트 8888)
  - WebRTC (포트 8889)
  - SRT (포트 8890)

주요 기능:
  - 프로토콜 간 자동 변환 (RTSP → HLS, RTSP → WebRTC)
  - 멀티캠 동시 처리 (Path 기반 스트림 구분)
  - 인증 (Internal, HTTP, JWT)
  - 녹화 (MP4 세그먼트)
  - 이벤트 훅 (on_publish, on_read, on_record)
  - API (REST)
  - Prometheus 메트릭 내보내기
```

---

## 2. 설치 및 실행

```bash
# 바이너리 다운로드
wget https://github.com/bluenviron/mediamtx/releases/latest/download/mediamtx_linux_amd64.tar.gz
tar -xzf mediamtx_linux_amd64.tar.gz

# 실행 (기본 설정)
./mediamtx

# Docker
docker run -d \
  --name mediamtx \
  --network host \
  -v $(pwd)/mediamtx.yml:/mediamtx.yml \
  bluenviron/mediamtx:latest

# Docker Compose
services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    network_mode: host        # RTSP/WebRTC UDP 범위 포트 때문에 host 권장
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
      - ./recordings:/recordings
    restart: unless-stopped
```

---

## 3. 핵심 설정 (mediamtx.yml)

```yaml
# mediamtx.yml

###############################################################################
# 프로토콜 활성화 및 포트 설정
###############################################################################

rtsp: yes
rtspAddress: :8554

rtmp: yes
rtmpAddress: :1935

hls: yes
hlsAddress: :8888
hlsAlwaysRemux: yes           # 구독자 없어도 항상 HLS 세그먼트 생성
hlsVariant: lowLatency        # 일반(6s) / lowLatency(~2s) / mpegts
hlsSegmentDuration: 2s
hlsPartDuration: 200ms        # LL-HLS 파트 단위
hlsSegmentMaxSize: 50M

webrtc: yes
webrtcAddress: :8889
webrtcLocalUDPAddress: :8189  # ICE 미디어 포트
webrtcICEServers2:
  - urls: [stun:stun.l.google.com:19302]

srt: yes
srtAddress: :8890

###############################################################################
# API 및 메트릭
###############################################################################

api: yes
apiAddress: :9997              # REST API 엔드포인트
metrics: yes
metricsAddress: :9998          # Prometheus /metrics

###############################################################################
# 인증
###############################################################################

authMethod: internal           # internal / http / jwt

# Internal 인증 (사용자 정의)
authInternalUsers:
  - user: admin
    pass: admin_password
    ips: []                    # [] = 모든 IP
    permissions:
      - action: publish
        path: ""               # 모든 경로
      - action: read
        path: ""
      - action: playback
        path: ""

  - user: viewer
    pass: viewer_password
    permissions:
      - action: read
        path: ""               # 읽기만 허용

###############################################################################
# 경로 (Path) 설정 — 각 스트림의 동작 정의
###############################################################################

paths:
  # ── 와일드카드: 모든 경로 기본값
  "~^.*$":
    source: publisher          # publish 연결을 기다림

  # ── IP 카메라 RTSP 소스 (pull 방식)
  cam1:
    source: rtsp://admin:pass@192.168.1.101:554/Streaming/Channels/101
    sourceProtocol: tcp
    sourceOnDemand: yes        # 구독자 있을 때만 카메라에 연결 (대역폭 절약)
    sourceOnDemandStartTimeout: 10s
    sourceOnDemandCloseAfter: 10s   # 구독자 끊기면 10초 후 카메라 연결 종료
    record: yes
    recordPath: /recordings/cam1/%Y-%m-%d_%H-%M-%S
    recordSegmentDuration: 1h
    recordDeleteAfter: 720h    # 30일 보존

  cam2:
    source: rtsp://admin:pass@192.168.1.102:554/Streaming/Channels/101
    sourceProtocol: tcp
    record: yes
    recordPath: /recordings/cam2/%Y-%m-%d_%H-%M-%S

  # ── FFmpeg onPublish 훅 — 발행 시 자동 처리
  live:
    runOnReady: >-
      ffmpeg -i rtsp://localhost:8554/$MTX_PATH
             -c:v libx264 -preset veryfast -tune zerolatency
             -f hls -hls_time 2 -hls_list_size 6
             /var/www/hls/$MTX_PATH/stream.m3u8 &
    runOnReadyRestart: yes

  # ── RTMP 수신 경로 (OBS Studio, IP 카메라 RTMP push)
  stream:
    source: publisher

###############################################################################
# 이벤트 훅 (글로벌)
###############################################################################

# 새 발행자 연결 시 (HTTP POST 알림)
runOnPublish: curl -s -X POST http://backend:8080/api/cameras/connected \
                -H "Content-Type: application/json" \
                -d '{"path":"$MTX_PATH","source":"$MTX_QUERY"}'
runOnPublishRestart: yes

# 발행 종료 시 (카메라 오프라인 감지)
runOnUnpublish: curl -s -X POST http://backend:8080/api/cameras/disconnected \
                  -d '{"path":"$MTX_PATH"}'
```

---

## 4. 클라이언트 접속 방법

```
[IP 카메라 — MediaMTX가 pull]
  카메라 RTSP → MediaMTX 자동 연결 (source: rtsp://...)

[OBS / FFmpeg — MediaMTX로 push]
  ffmpeg -re -i input.mp4 -c copy -f rtsp rtsp://localhost:8554/stream
  ffmpeg -re -i input.mp4 -c copy -f flv rtmp://localhost:1935/stream

[VMS / RTSP 클라이언트 — 재배포 수신]
  ffplay rtsp://mediamtx-host:8554/cam1
  vlc rtsp://mediamtx-host:8554/cam1

[웹 브라우저 — HLS]
  http://mediamtx-host:8888/cam1/index.m3u8

[웹 브라우저 — WebRTC (초저지연)]
  http://mediamtx-host:8889/cam1  ← MediaMTX 내장 WebRTC 플레이어

[SRT 클라이언트]
  ffplay srt://mediamtx-host:8890?streamid=read:cam1
```

---

## 5. 다중 카메라 아키텍처

```
[IP 카메라들]
  cam01 (RTSP) ──┐
  cam02 (RTSP) ──┤
  cam03 (RTSP) ──┤──▶ [MediaMTX]
  cam04 (RTMP) ──┘         │
                    ┌───────┴──────────────────┐
                    ▼              ▼            ▼
               [RTSP 재배포]   [HLS 배포]   [WebRTC 배포]
               NVR / VMS      웹 브라우저   모바일 앱
               8554 포트       8888 포트     8889 포트

# Path 구조 예시
RTSP:    rtsp://mediamtx:8554/cam01
HLS:     http://mediamtx:8888/cam01/index.m3u8
WebRTC:  http://mediamtx:8889/cam01
```

### Docker Compose — 다중 카메라 전체 스택

```yaml
version: "3.8"

services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    network_mode: host
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
      - recordings:/recordings
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:9997/v3/paths/list"]
      interval: 30s
      retries: 3

  # Nginx — HLS 파일 서빙 + SSL 종단
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl
      - recordings:/var/www/hls:ro

  # 백엔드 — 이벤트 수신 + 카메라 관리
  backend:
    image: myapp/backend:latest
    environment:
      - MEDIAMTX_API=http://mediamtx:9997

volumes:
  recordings:
```

---

## 6. REST API

```bash
# 전체 경로(스트림) 목록
curl http://mediamtx:9997/v3/paths/list

# 응답 예시:
# {
#   "items": [
#     {"name": "cam1", "ready": true,
#      "readers": [{"type": "webRTCSession", "id": "..."}],
#      "source": {"type": "rtspSession"}}
#   ]
# }

# 특정 경로 정보
curl http://mediamtx:9997/v3/paths/get/cam1

# 활성 WebRTC 세션 목록
curl http://mediamtx:9997/v3/webrtcsessions/list

# 활성 RTSP 세션 목록
curl http://mediamtx:9997/v3/rtspsessions/list

# 녹화 목록
curl http://mediamtx:9997/v3/recordings/list

# 특정 경로 강제 종료
curl -X DELETE http://mediamtx:9997/v3/rtspsessions/kick/SESSION_ID
```

---

## 7. 녹화 (Recording)

```yaml
# mediamtx.yml — 녹화 설정
paths:
  cam1:
    source: rtsp://cam1-ip/stream
    record: yes
    recordPath: /recordings/cam1/%Y-%m-%d/%H/%Y%m%d_%H%M%S  # 날짜/시간 디렉토리
    recordFormat: mp4          # mp4 또는 fmp4 (fragmented, 장애 안전)
    recordPartDuration: 200ms  # fmp4 파트 단위 (장애 시 손실 최소화)
    recordSegmentDuration: 30m # 파일 분할 단위 (30분)
    recordDeleteAfter: 720h    # 30일 후 자동 삭제 (0 = 삭제 안 함)
```

```
녹화 파일 구조:
  /recordings/
  ├── cam1/
  │   ├── 2026-06-16/
  │   │   ├── 14/
  │   │   │   ├── 20260616_140000.mp4
  │   │   │   ├── 20260616_143000.mp4
  │   │   │   └── 20260616_150000.mp4
  │   │   └── 15/
  │   └── 2026-06-17/
  └── cam2/

재생 (녹화 파일에서):
  ffplay /recordings/cam1/2026-06-16/14/20260616_140000.mp4
  또는 HLS로 서빙:
  ffmpeg -re -i /recordings/.../clip.mp4 -c copy -f hls ...
```

---

## 8. Prometheus 메트릭

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: mediamtx
    static_configs:
      - targets: [mediamtx:9998]
```

```
# 주요 메트릭
mediamtx_hlssegments_total        # 생성된 HLS 세그먼트 수
mediamtx_rtspconnections_total    # 총 RTSP 연결 수
mediamtx_webrtcsessions_total     # 총 WebRTC 세션 수
mediamtx_bytes_received_total     # 수신 바이트
mediamtx_bytes_sent_total         # 송신 바이트

# Grafana 대시보드: ID 19592 (MediaMTX)
```

---

## 9. CCTV 웹 뷰어 구현 (HLS.js)

```html
<!-- 웹 브라우저에서 HLS 재생 -->
<!DOCTYPE html>
<html>
<head>
  <title>CCTV 모니터링</title>
  <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
</head>
<body>
  <video id="cam1" controls muted autoplay width="640"></video>

  <script>
    const video = document.getElementById('cam1');
    const src = 'http://mediamtx:8888/cam1/index.m3u8';

    if (Hls.isSupported()) {
      const hls = new Hls({ lowLatencyMode: true });
      hls.loadSource(src);
      hls.attachMedia(video);
      hls.on(Hls.Events.MANIFEST_PARSED, () => video.play());
    } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
      // Safari 네이티브 HLS
      video.src = src;
      video.play();
    }
  </script>
</body>
</html>
```

### WebRTC 뷰어 (MediaMTX 내장)

```
# MediaMTX는 http://host:8889/{path} 에서 내장 WebRTC 플레이어 제공
# 별도 코딩 없이 바로 브라우저에서 초저지연 시청 가능

# 커스텀 WebRTC 구현 시 WHEPClient API 사용:
const client = new WHEPClient();
await client.setOffer(await pc.createOffer());
```

---

## 10. RTSP 미러링 — 현장 대역폭 보호

### 문제: 직접 RTSP 풀 시 대역폭 폭증

```
[미러링 없을 때]
  현장 카메라 (4 Mbps × 10대)
       │
  ┌────┤ ──────────▶ VMS 클라이언트 (4 Mbps)
  │    │ ──────────▶ Spring 백엔드  (4 Mbps)
  │    │ ──────────▶ AI 분석 서버   (4 Mbps)
  │    │ ──────────▶ 관제실 모니터  (4 Mbps)
  │    │ ──────────▶ 웹 뷰어        (4 Mbps)
  └────┘
  현장 → 외부 대역폭: 4 Mbps × 5 소비자 × 10대 = 200 Mbps 필요

  결과: 대역폭 포화 → 스트림 끊김, 카메라 자체 응답 불가,
        카메라 인증 정보가 여러 시스템에 분산 노출

[미러링 적용 후]
  현장 카메라 (4 Mbps × 10대)
       │  (각 카메라를 MediaMTX가 1회만 풀)
  [현장 Edge MediaMTX]
       │  (단일 업링크 스트림만 외부로 나감)
  [중앙 MediaMTX]
       │
  ├──▶ VMS 클라이언트
  ├──▶ Spring 백엔드
  ├──▶ AI 분석 서버
  ├──▶ 관제실 모니터
  └──▶ 웹 뷰어

  현장 → 외부 대역폭: 4 Mbps × 10대 = 40 Mbps (소비자 수 무관)
```

---

### 구조 1 — 현장 Edge + 중앙 Center (2-Tier)

```
[현장 네트워크]
  cam01~cam10 (H.265, 4 Mbps)
         │ 각 카메라에서 1회 pull
  [Edge MediaMTX]  ← 현장 서버/NUC/라즈베리파이
  포트 8554 내부 전용
         │
         │ 단일 업스트림 RTSP (카메라 수 × 4 Mbps = 40 Mbps)
         │ (현장 방화벽: 8554만 외부 허용)
         ▼
  [Center MediaMTX]  ← 사무소/클라우드
  포트 8554 (인증 필수)
         │
  ┌──────┴────────────────────────────┐
  ▼            ▼              ▼        ▼
 VMS          HLS          WebRTC   Spring
 (RTSP)      (8888)        (8889)   Backend
```

**Edge MediaMTX** (`edge/mediamtx.yml`):

```yaml
rtsp: yes
rtspAddress: :8554

# 인증 없음 — 현장 내부망 전용 (방화벽으로 보호)
authMethod: ""

paths:
  # 카메라 → Edge MediaMTX pull (내부)
  cam01:
    source: rtsp://admin:pass@192.168.1.101:554/Streaming/Channels/101
    sourceProtocol: tcp
    sourceOnDemand: yes               # 구독자 있을 때만 카메라 연결

  cam02:
    source: rtsp://admin:pass@192.168.1.102:554/Streaming/Channels/101
    sourceProtocol: tcp
    sourceOnDemand: yes

  # ... cam03~cam10 동일 패턴

# 카메라 인증 정보는 Edge 서버에만 존재
# 외부에서는 Edge RTSP 주소만 알면 됨 (카메라 직접 접근 불가)
```

**Center MediaMTX** (`center/mediamtx.yml`):

```yaml
rtsp: yes
rtspAddress: :8554

hls: yes
hlsAddress: :8888
hlsAlwaysRemux: yes
hlsVariant: lowLatency
hlsSegmentDuration: 2s

webrtc: yes
webrtcAddress: :8889

api: yes
apiAddress: :9997

# 중앙 서버는 인증 필수 (외부 클라이언트 접근)
authMethod: internal
authInternalUsers:
  - user: vms
    pass: vms_secure_pass
    permissions:
      - action: read
        path: ""
  - user: backend
    pass: backend_secure_pass
    permissions:
      - action: read
        path: ""

paths:
  # Edge MediaMTX → Center MediaMTX pull
  cam01:
    source: rtsp://edge-server:8554/cam01   # Edge에서 가져옴
    sourceProtocol: tcp
    sourceOnDemand: yes
    # Center도 구독자 있을 때만 Edge 연결 (2단계 수요 기반)

  cam02:
    source: rtsp://edge-server:8554/cam02
    sourceProtocol: tcp
    sourceOnDemand: yes
```

---

### 구조 2 — 현장 Edge만 (소규모, 단순)

```
[현장 내부]
  cam01~cam10
       │ (각 1회 pull)
  [Edge MediaMTX]  ← 현장 내부망
       │
  ┌────┴─────────────────────┐
  ▼          ▼               ▼
 내부 VMS  AI 서버         현장 모니터
 (RTSP)    (RTSP)          (HLS)

[외부 접근 — VPN을 통해서만]
  개발자/관리자 → VPN → Edge MediaMTX RTSP
```

```yaml
# edge/mediamtx.yml (단순 구조)
rtsp: yes
rtspAddress: :8554

hls: yes           # 현장 내부 웹 뷰어용
hlsAddress: :8888

authMethod: internal
authInternalUsers:
  - user: internal
    pass: changeme
    permissions:
      - action: read
        path: ""
  - user: edge-publisher    # Edge가 카메라 대신 발행할 경우
    permissions:
      - action: publish
        path: ""

paths:
  "~^cam[0-9]+$":           # cam0~cam9999 패턴 자동 처리
    source: publisher        # 또는 각 카메라 URL 직접 지정
    sourceOnDemand: yes
    record: yes
    recordPath: /recordings/%path/%Y%m%d_%H%M%S
    recordSegmentDuration: 1h
    recordDeleteAfter: 720h
```

---

### `sourceOnDemand` — 수요 기반 연결로 추가 절약

```yaml
paths:
  cam01:
    source: rtsp://cam1-ip/stream
    sourceOnDemand: yes

    # 구독자가 없으면 카메라 연결 자동 종료
    sourceOnDemandCloseAfter: 30s   # 마지막 구독자 이탈 후 30초 뒤 닫음

    # 구독자가 나타나면 카메라 재연결
    sourceOnDemandStartTimeout: 10s # 연결 시도 타임아웃
```

```
효과:
  야간/비업무 시간에 아무도 보지 않으면 카메라 연결 자동 종료
  → 현장 업링크 대역폭 = 0 Mbps (녹화 스트림 제외)
  → 카메라 인터페이스 부하 감소

주의:
  VMS나 AI 분석 서버가 24시간 연결하면 sourceOnDemand 효과 없음
  → 녹화/AI는 별도 sourceOnDemand: no 경로 사용 권장
```

---

### 보안 강화 — 카메라 직접 접근 차단

```
[방화벽 규칙]
  현장 방화벽:
    ✅ 허용: Edge MediaMTX → 카메라 554/udp, 554/tcp (내부망)
    ✅ 허용: 외부 → Edge MediaMTX 8554/tcp (VPN or 특정 IP만)
    ❌ 차단: 외부 → 카메라 직접 접근 (554 포트 외부 차단)

  보안 효과:
    카메라 admin:pass 자격증명 → Edge MediaMTX만 알고 있음
    외부 시스템은 MediaMTX 인증(별도 계정)으로만 접근
    카메라 펌웨어 취약점 → 외부에서 직접 익스플로잇 불가
```

```yaml
# Edge MediaMTX에서 외부 클라이언트도 인증 요구
authMethod: internal
authInternalUsers:
  - user: center        # 중앙 서버용
    pass: center_pass
    ips: [10.0.0.5]     # 중앙 서버 IP만 허용
    permissions:
      - action: read
        path: ""

  - user: vpn-admin     # 관리자 VPN 접근
    pass: admin_pass
    ips: [172.16.0.0/12]  # VPN 대역
    permissions:
      - action: read
        path: ""

# 기본: 모든 인증 실패 차단 (authMethod: internal 설정 시 자동)
```

---

### 대역폭 절감 계산 예시

```
[환경]
  카메라 20대 × H.265 4 Mbps
  소비자: VMS(1) + AI서버(1) + 웹뷰어(최대 50명) + 백엔드(1)

[미러링 없음]
  카메라 직접 접근 시:
    VMS:     20 × 4 Mbps = 80 Mbps
    AI:      20 × 4 Mbps = 80 Mbps
    웹뷰어:  (HLS 개별 계산, 하지만 카메라 RTSP 라면) 200 Mbps+
    총 현장 업링크: 360 Mbps+ 필요 → 일반 인터넷 회선 포화

[Edge MediaMTX 적용]
  현장 → Center:   20 × 4 Mbps = 80 Mbps (소비자 수 고정)
  Center → 소비자: Center 서버 대역폭 (현장 무관)
  현장 업링크: 80 Mbps (소비자 50명이 되어도 동일)

[sourceOnDemand + 야간 스케줄]
  업무 시간(9-18시): 80 Mbps
  야간(18-9시): 녹화 스트림만 → 20~40 Mbps (AI 분석 중단 시 더 절감)
```

---

### Docker Compose — Edge + Center 배포

```yaml
# edge/docker-compose.yml (현장 서버)
version: "3.8"
services:
  mediamtx-edge:
    image: bluenviron/mediamtx:latest
    network_mode: host
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
      - /mnt/recordings:/recordings   # 현장 NAS 마운트
    restart: unless-stopped
    environment:
      - MTX_LOGLEVEL=warn

---
# center/docker-compose.yml (중앙 서버)
version: "3.8"
services:
  mediamtx-center:
    image: bluenviron/mediamtx:latest
    network_mode: host
    volumes:
      - ./mediamtx.yml:/mediamtx.yml
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports: ["443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl
    # HLS를 HTTPS로 서빙
```

---

## 11. PTZ 조작용 WebRTC 온디맨드 세션

### 문제: PTZ 조작 시 지연이 사용자 경험을 망침

```
[프로토콜별 피드백 지연]
  HLS (일반):        6~30초  ← Pan 했는데 화면이 30초 후 반응
  HLS (LL-HLS):      2~5초   ← 여전히 어색함
  RTSP/VMS:          0.5~2초 ← 데스크톱 앱에선 OK
  WebRTC:            100~300ms ← PTZ 조작에 적합한 유일한 선택

결론:
  평상시 모니터링: HLS (서버 부하 낮음, CDN 가능)
  PTZ 조작 중:    WebRTC (초저지연, 조작-반응 실시간성)
  → PTZ 세션 동안만 WebRTC를 열고, 완료 후 자동 종료
```

### MediaMTX 설정 — PTZ 전용 경로

```yaml
# mediamtx.yml

paths:
  # ── 일반 모니터링 경로 (HLS, 상시)
  cam01:
    source: rtsp://192.168.1.101:554/Streaming/Channels/101
    sourceProtocol: tcp
    record: yes
    recordPath: /recordings/cam01/%Y%m%d_%H%M%S
    recordSegmentDuration: 1h

  # ── PTZ 조작 전용 경로 (WebRTC 온디맨드)
  # 명명 규칙: {camId}-ptz
  cam01-ptz:
    source: rtsp://192.168.1.101:554/Streaming/Channels/101
    sourceProtocol: tcp

    # 구독자 없으면 카메라 연결 즉시 해제
    sourceOnDemand: yes
    sourceOnDemandStartTimeout: 5s
    sourceOnDemandCloseAfter: 60s   # PTZ 완료 후 최대 60초 유지

    # PTZ 세션: 인증된 사용자만 (일반 뷰어보다 엄격)
    # → Spring 백엔드가 WebRTC 세션 생성 API를 게이트로 제어

    # 이벤트 훅: 세션 열림/닫힘 백엔드 알림
    runOnReady: >-
      curl -s -X POST http://backend:8080/api/ptz/session/opened
           -d '{"path":"$MTX_PATH"}'
    runOnNotReady: >-
      curl -s -X POST http://backend:8080/api/ptz/session/closed
           -d '{"path":"$MTX_PATH"}'
```

### Spring 백엔드 — PTZ 세션 관리

```java
@RestController
@RequestMapping("/api/ptz")
@RequiredArgsConstructor
@Slf4j
public class PtzSessionController {

    private final MediaMtxClient mediaMtxClient;
    private final PtzControlService ptzService;
    private final Map<String, ScheduledFuture<?>> sessionTimers = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);

    private static final Duration PTZ_SESSION_TIMEOUT = Duration.ofHours(1);

    /**
     * PTZ 세션 시작 — WebRTC URL 반환 + 1시간 타이머 등록.
     * 클라이언트는 반환된 webrtcUrl로 MediaMTX WebRTC 플레이어에 직접 연결.
     */
    @PostMapping("/session/start")
    @PreAuthorize("hasRole('PTZ_OPERATOR')")
    public PtzSessionResponse startPtzSession(
            @RequestParam String cameraId,
            @AuthenticationPrincipal UserDetails user) {

        String ptzPath = cameraId + "-ptz";
        String webrtcUrl = "http://mediamtx:8889/" + ptzPath;

        // MediaMTX는 sourceOnDemand: yes 이므로 첫 구독자가 연결하면 카메라 연결 자동 시작

        // 1시간 뒤 자동 종료 타이머 등록
        cancelExistingTimer(ptzPath);
        ScheduledFuture<?> timer = scheduler.schedule(
            () -> forceClosePtzSession(cameraId, ptzPath),
            PTZ_SESSION_TIMEOUT.toSeconds(),
            TimeUnit.SECONDS
        );
        sessionTimers.put(ptzPath, timer);

        log.info("PTZ session started: {} by {}", cameraId, user.getUsername());
        return new PtzSessionResponse(webrtcUrl, Instant.now().plus(PTZ_SESSION_TIMEOUT));
    }

    /**
     * PTZ 조작 — 세션 활성 상태에서만 허용.
     * 조작할 때마다 타이머 리셋 (활동 감지).
     */
    @PostMapping("/move")
    @PreAuthorize("hasRole('PTZ_OPERATOR')")
    public void ptzMove(@RequestParam String cameraId,
                        @RequestParam double panSpeed,
                        @RequestParam double tiltSpeed) {
        String ptzPath = cameraId + "-ptz";

        if (!sessionTimers.containsKey(ptzPath)) {
            throw new IllegalStateException("PTZ session not active for " + cameraId);
        }

        ptzService.continuousMove(resolveCameraIp(cameraId), panSpeed, tiltSpeed);

        // 활동 시 타이머 리셋 (1시간 연장)
        resetSessionTimer(cameraId, ptzPath);
    }

    /** PTZ 세션 수동 종료 */
    @DeleteMapping("/session/{cameraId}")
    @PreAuthorize("hasRole('PTZ_OPERATOR')")
    public void stopPtzSession(@PathVariable String cameraId) {
        forceClosePtzSession(cameraId, cameraId + "-ptz");
    }

    private void forceClosePtzSession(String cameraId, String ptzPath) {
        // MediaMTX API: 해당 경로의 모든 WebRTC 세션 강제 종료
        mediaMtxClient.kickAllReadersFromPath(ptzPath);
        sessionTimers.remove(ptzPath);
        log.info("PTZ session auto-closed: {} (timeout or manual)", cameraId);
    }

    private void resetSessionTimer(String cameraId, String ptzPath) {
        cancelExistingTimer(ptzPath);
        ScheduledFuture<?> timer = scheduler.schedule(
            () -> forceClosePtzSession(cameraId, ptzPath),
            PTZ_SESSION_TIMEOUT.toSeconds(),
            TimeUnit.SECONDS
        );
        sessionTimers.put(ptzPath, timer);
    }

    private void cancelExistingTimer(String ptzPath) {
        ScheduledFuture<?> existing = sessionTimers.get(ptzPath);
        if (existing != null) existing.cancel(false);
    }
}
```

### MediaMTX API — 세션 강제 종료

```java
@Service
@RequiredArgsConstructor
public class MediaMtxClient {

    private final RestTemplate restTemplate;

    @Value("${mediamtx.api-url:http://mediamtx:9997}")
    private String apiUrl;

    /** 특정 경로의 모든 WebRTC 구독자 강제 종료 */
    public void kickAllReadersFromPath(String path) {
        // 활성 WebRTC 세션 조회
        var sessions = restTemplate.getForObject(
            apiUrl + "/v3/webrtcsessions/list", WebRtcSessionList.class);

        if (sessions == null) return;

        sessions.getItems().stream()
            .filter(s -> path.equals(s.getPath()))
            .forEach(s -> restTemplate.delete(
                apiUrl + "/v3/webrtcsessions/kick/" + s.getId()));
    }

    /** 특정 경로의 상태 확인 (구독자 수 포함) */
    public PathInfo getPathInfo(String path) {
        return restTemplate.getForObject(
            apiUrl + "/v3/paths/get/" + path, PathInfo.class);
    }
}
```

### 프론트엔드 — WebRTC 뷰어 + PTZ 컨트롤 패널

```javascript
// PTZ 세션 시작 → WebRTC 연결
async function startPtzSession(cameraId) {
    // 1. 백엔드에서 세션 시작 + WebRTC URL 획득
    const resp = await fetch(`/api/ptz/session/start?cameraId=${cameraId}`, {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${token}` }
    });
    const { webrtcUrl, expiresAt } = await resp.json();

    // 2. MediaMTX 내장 WebRTC 플레이어로 초저지연 스트림 연결
    // webrtcUrl = "http://mediamtx:8889/cam01-ptz"
    document.getElementById('ptz-frame').src = webrtcUrl;

    // 3. 만료 타이머 UI 표시
    showCountdown(new Date(expiresAt));
}

// PTZ 조작 — 버튼 누르는 동안 연속 이동
document.getElementById('btn-left').addEventListener('mousedown', () => {
    fetch(`/api/ptz/move?cameraId=cam01&panSpeed=-0.5&tiltSpeed=0`, {
        method: 'POST', headers: { 'Authorization': `Bearer ${token}` }
    });
});
document.getElementById('btn-left').addEventListener('mouseup', () => {
    fetch(`/api/ptz/move?cameraId=cam01&panSpeed=0&tiltSpeed=0`, {
        method: 'POST'  // Stop
    });
});
```

### PTZ-WebRTC 흐름 요약

```
사용자 요청 "/api/ptz/session/start"
       │
Spring → ptzPath: "cam01-ptz" WebRTC URL 반환
       │ 1시간 타이머 시작
       ▼
브라우저 → MediaMTX :8889/cam01-ptz (WebRTC)
       │ (첫 구독자 → sourceOnDemand 트리거 → 카메라 RTSP 연결)
       ▼
초저지연 영상 수신 (~200ms) ← PTZ 조작 즉시 반영
       │
사용자: 방향키 클릭 → /api/ptz/move → ONVIF SOAP → 카메라 물리 이동
       │ (조작할 때마다 타이머 리셋)
       ▼
1시간 무조작 또는 수동 종료:
  Spring → MediaMTX API kick → WebRTC 세션 강제 해제
  sourceOnDemandCloseAfter: 60s 후 카메라 연결도 자동 종료
  → GPU/CPU/대역폭 반납
```

---

## 12. 관련
- [[Streaming-Protocols]] — RTSP, HLS, WebRTC, SRT 프로토콜 심화, ONVIF PTZ
- [[FFmpeg]] — MediaMTX와 연동하는 트랜스코딩 파이프라인
- [[Video-Fundamentals]] — 코덱, 색상 포맷, 비트레이트 설계
- [[Spring-CCTV]] — Spring 기반 JavaCV vs FFmpeg 아키텍처
