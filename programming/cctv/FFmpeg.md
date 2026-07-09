---
tags:
  - cctv
  - ffmpeg
  - transcoding
  - video
  - streaming
created: 2026-06-16
---

# FFmpeg — 영상 처리·트랜스코딩·스트리밍

> [!summary] 한 줄 요약
> FFmpeg는 멀티미디어 처리의 만능 스위스 아미 나이프. CCTV 파이프라인에서 **코덱 변환·스트림 복사·RTSP 수신→HLS 배포·영상 필터링·하드웨어 가속**까지 처리한다.

---

## 1. FFmpeg 아키텍처

```
입력 처리                   변환·처리                  출력
─────────                   ─────────                  ────
Demuxer                     Decoder                    Encoder
 MP4, TS, MKV     패킷 ──▶ H.264→raw 프레임 ──▶ H.265 인코딩 ──▶ Muxer
 RTSP, RTMP               ──────────────────                        MP4/HLS/RTMP
 SRT, HLS                 Filter Graph
                            scale, crop, drawtext,
                            overlay, fps, etc.

[스트림 복사 모드 — 디코딩/인코딩 없음]
  ffmpeg -i input -c copy output
  → CPU 부하 최소, 화질 손실 없음, 코덱 변환 불필요 시 최선
```

---

## 2. 기본 사용법

### 비디오 트랜스코딩

```bash
# H.264 → H.265 변환 (저장 비용 절감)
ffmpeg -i input.mp4 \
  -c:v libx265 -crf 28 -preset medium \
  -c:a copy \
  output_h265.mp4

# CRF (Constant Rate Factor): 낮을수록 고화질, 높을수록 소파일
#   H.264: 18(고품질) ~ 28(중간) ~ 35(저품질)
#   H.265: 24(고품질) ~ 28(중간) ~ 35(저품질)

# 해상도 변환 (1080p → 720p)
ffmpeg -i input.mp4 \
  -vf scale=1280:720 \
  -c:v libx264 -crf 23 -preset fast \
  output_720p.mp4

# 특정 시간 구간만 추출
ffmpeg -ss 00:01:30 -i input.mp4 -t 60 -c copy clip.mp4
#   -ss 위치 (입력 앞: 빠른 탐색, 입력 뒤: 정확한 탐색)
#   -t  지속 시간 (초)

# 스트림 정보 확인 (인코딩 없음)
ffprobe -v quiet -print_format json -show_streams input.mp4
```

### 스트림 복사 (코덱 변환 없음)

```bash
# 컨테이너만 변경 (MP4 → MKV, 거의 즉시 완료)
ffmpeg -i input.mp4 -c copy output.mkv

# RTSP 수신 → MP4 파일 저장 (1시간)
ffmpeg -rtsp_transport tcp \
  -i rtsp://admin:pass@192.168.1.100/stream1 \
  -c copy -t 3600 \
  recording_$(date +%Y%m%d_%H%M%S).mp4

# 특정 스트림만 추출 (비디오만, 오디오 제거)
ffmpeg -i input.mp4 -c:v copy -an video_only.mp4

# RTSP → HLS 변환 (스트림 복사)
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream \
  -c copy -f hls \
  -hls_time 6 \
  -hls_list_size 10 \
  -hls_flags delete_segments \
  /var/www/hls/cam1/stream.m3u8
```

---

## 3. 필터그래프 (Filter Graph)

### 기본 필터 체인

```bash
# 크롭 + 스케일 + 텍스트 오버레이
ffmpeg -i input.mp4 \
  -vf "crop=1280:720:320:180, scale=640:360, \
       drawtext=text='CAM-01':x=10:y=10:fontsize=24:fontcolor=white" \
  output.mp4

# 복잡한 필터 (사각 모자이크 — 개인정보 보호)
ffmpeg -i input.mp4 \
  -vf "delogo=x=100:y=50:w=200:h=150:show=0" \
  output.mp4

# 밝기·대비 조정 (야간 CCTV 보정)
ffmpeg -i input.mp4 \
  -vf "eq=brightness=0.06:saturation=2:contrast=1.2" \
  output.mp4

# 프레임레이트 변환 (30fps → 15fps, 저장 절약)
ffmpeg -i input.mp4 \
  -vf "fps=15" \
  -c:v libx264 -crf 23 \
  output_15fps.mp4
```

### 복합 필터그래프 — 멀티캠 그리드

```bash
# 4채널 2×2 그리드 합성
ffmpeg \
  -i rtsp://cam1/stream -i rtsp://cam2/stream \
  -i rtsp://cam3/stream -i rtsp://cam4/stream \
  -filter_complex "
    [0:v]scale=960:540[v0];
    [1:v]scale=960:540[v1];
    [2:v]scale=960:540[v2];
    [3:v]scale=960:540[v3];
    [v0][v1]hstack=inputs=2[top];
    [v2][v3]hstack=inputs=2[bottom];
    [top][bottom]vstack=inputs=2[out]
  " \
  -map "[out]" -c:v libx264 -crf 23 \
  -f rtsp rtsp://localhost:8554/grid
```

---

## 4. RTSP 스트리밍 패턴

### RTSP 수신 (IP 카메라)

```bash
# 실시간 재생 (ffplay)
ffplay -rtsp_transport tcp rtsp://admin:pass@192.168.1.100/stream1

# 연속 녹화 (시간 기반 파일 분할)
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -c copy \
  -f segment -segment_time 3600 -reset_timestamps 1 \
  -strftime 1 \
  "/recordings/cam1/%Y%m%d_%H%M%S.mp4"

# 스트림 재접속 옵션 (카메라 재시작 대응)
ffmpeg -rtsp_transport tcp \
  -reconnect 1 -reconnect_at_eof 1 -reconnect_streamed 1 \
  -reconnect_delay_max 30 \
  -i rtsp://cam1/stream1 \
  -c copy output.mp4
```

### RTSP → HLS 라이브 스트리밍

```bash
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -c:v libx264 -preset veryfast -tune zerolatency \
  -b:v 2000k -maxrate 2500k -bufsize 5000k \
  -g 30 -sc_threshold 0 \
  -f hls \
  -hls_time 2 \                    # 2초 세그먼트 (지연 최소화)
  -hls_list_size 6 \               # 플레이리스트에 유지할 세그먼트 수
  -hls_flags delete_segments+append_list \
  -hls_segment_filename "/hls/cam1/seg%05d.ts" \
  /hls/cam1/stream.m3u8
```

### RTSP → RTMP (유튜브/트위치 라이브)

```bash
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -c:v libx264 -preset veryfast \
  -b:v 3000k -maxrate 3500k -bufsize 7000k \
  -c:a aac -b:a 128k \
  -f flv rtmp://a.rtmp.youtube.com/live2/STREAM_KEY
```

---

## 5. 하드웨어 가속

### NVIDIA NVENC/NVDEC

```bash
# GPU 가속 인코딩 (소프트웨어 대비 CPU 10~20배 절감)
ffmpeg -hwaccel cuda -hwaccel_output_format cuda \
  -i input.mp4 \
  -c:v h264_nvenc \
  -preset p4 \          # p1(빠름,저품질) ~ p7(느림,고품질)
  -cq 23 \              # CRF 유사 (0=최고, 51=최저)
  -b:v 0 \
  output_nvenc.mp4

# H.265 NVENC
ffmpeg -hwaccel cuda \
  -i input.mp4 \
  -c:v hevc_nvenc -preset p4 -cq 28 \
  output_hevc.mp4

# 지원 인코더 확인
ffmpeg -hide_banner -encoders | grep nvenc
```

### Intel QSV (Quick Sync Video)

```bash
# 인텔 내장 GPU 가속
ffmpeg -hwaccel qsv -hwaccel_device /dev/dri/renderD128 \
  -i input.mp4 \
  -c:v h264_qsv \
  -preset medium \
  -b:v 2000k \
  output_qsv.mp4
```

### VAAPI (Linux 범용 하드웨어 가속)

```bash
ffmpeg -hwaccel vaapi \
  -hwaccel_device /dev/dri/renderD128 \
  -hwaccel_output_format vaapi \
  -i input.mp4 \
  -vf 'hwupload,scale_vaapi=1280:720' \
  -c:v h264_vaapi \
  -qp 23 \
  output_vaapi.mp4
```

### Apple VideoToolbox (macOS)

```bash
# M1/M2/M3 하드웨어 가속
ffmpeg -i input.mp4 \
  -c:v h264_videotoolbox \
  -b:v 3000k \
  output_vt.mp4
```

---

## 6. CCTV 운영 패턴

### 연속 녹화 스크립트

```bash
#!/bin/bash
# continuous_record.sh — 장애 자동 재시작

CAMERA_URL="rtsp://admin:pass@192.168.1.100/stream1"
OUTPUT_DIR="/recordings/cam01"
SEGMENT_HOURS=1

mkdir -p "$OUTPUT_DIR"

while true; do
    ffmpeg \
        -rtsp_transport tcp \
        -reconnect 1 -reconnect_at_eof 1 -reconnect_streamed 1 \
        -reconnect_delay_max 30 \
        -i "$CAMERA_URL" \
        -c copy \
        -f segment \
        -segment_time $((SEGMENT_HOURS * 3600)) \
        -reset_timestamps 1 \
        -strftime 1 \
        "$OUTPUT_DIR/%Y%m%d_%H%M%S.mp4"

    echo "[$(date)] FFmpeg exited with code $?, restarting in 5s..."
    sleep 5
done
```

### 모션 감지 (임계값 기반 트리밍)

```bash
# 움직임 있는 구간만 추출 (저장 절약)
ffmpeg -i input.mp4 \
  -vf "select='gt(scene,0.1)',setpts=N/FRAME_RATE/TB" \
  -af "aselect='gt(scene,0.1)',asetpts=N/SR/TB" \
  motion_clips.mp4

# scene 값: 0=변화없음, 1=완전히 다른 프레임 (0.1 = 10% 변화)
```

### 썸네일·스냅샷 추출

```bash
# 10초마다 JPEG 스냅샷
ffmpeg -i input.mp4 \
  -vf fps=1/10 \
  /snapshots/thumb%04d.jpg

# 특정 시점 단일 프레임
ffmpeg -ss 00:05:30 -i input.mp4 \
  -vframes 1 -q:v 2 \
  snapshot.jpg

# RTSP 현재 프레임 즉시 캡처
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -vframes 1 -q:v 2 \
  "snapshot_$(date +%Y%m%d_%H%M%S).jpg"
```

### 저장 공간 관리

```bash
# 90일 이상 된 녹화 파일 삭제
find /recordings -name "*.mp4" -mtime +90 -delete

# 디스크 사용률 90% 초과 시 오래된 파일부터 삭제
THRESHOLD=90
USAGE=$(df /recordings | awk 'NR==2 {print $5}' | tr -d '%')
if [ "$USAGE" -gt "$THRESHOLD" ]; then
    find /recordings -name "*.mp4" -printf '%T+ %p\n' | \
        sort | head -10 | awk '{print $2}' | xargs rm -f
fi
```

---

## 7. 픽셀 포맷 변환

```bash
# YUV420p → NV12 (하드웨어 인코더 입력 포맷 맞춤)
ffmpeg -i input.mp4 \
  -vf format=nv12 \
  -c:v h264_nvenc output.mp4

# NV12 → RGB24 (OpenCV 처리용)
ffmpeg -i input.mp4 \
  -vf format=rgb24 \
  -f rawvideo pipe:1 | \
  python3 process_frames.py

# 파이프로 raw 프레임 전달
ffmpeg -i rtsp://cam1/stream \
  -f rawvideo -pix_fmt bgr24 pipe:1
# → Python/OpenCV: cap = cv2.VideoCapture(ffmpeg_pipe)
```

---

## 8. 유용한 진단 명령

```bash
# 스트림 정보
ffprobe -v error -select_streams v:0 \
  -show_entries stream=codec_name,width,height,r_frame_rate,bit_rate \
  -of default=noprint_wrappers=1 input.mp4

# 오디오 정보
ffprobe -v error -select_streams a:0 \
  -show_entries stream=codec_name,sample_rate,channels,bit_rate \
  -of default=noprint_wrappers=1 input.mp4

# 패킷 타임스탬프 확인 (깨진 스트림 디버깅)
ffprobe -v quiet -show_packets -select_streams v:0 \
  -print_format json input.mp4 | head -100

# 실시간 통계 (인코딩 속도, 프레임 수)
ffmpeg ... -progress pipe:2 2>&1 | grep -E "fps|speed|time"
```

---

## 9. H.265·H.266 → H.264 강제 변환 (브라우저 호환)

### 왜 H.264로 변환해야 하는가

```
[브라우저 코덱 지원 현황]
  H.264 (AVC):  Chrome/Firefox/Safari/Edge → ✅ 100% 지원
  H.265 (HEVC): Safari 11+/Edge ✅, Chrome/Firefox ❌
  H.266 (VVC):  현재 모든 브라우저 미지원 ❌
  AV1:          Chrome 70+/Firefox 67+/Edge ✅, Safari 16+ ✅ (점진적)

[결론]
  범용 웹 스트리밍 = H.264 강제 변환 필수
  H.265 CCTV 카메라 → H.264 트랜스코딩 → HLS/WebRTC 배포
```

### 트랜스코딩 파이프라인

```bash
# H.265 카메라 → H.264 HLS 변환
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -c:v libx264 \          # H.265 입력 → H.264 소프트웨어 인코딩
  -preset veryfast \       # 실시간 인코딩 (ultrafast: 품질↓, medium: 지연↑)
  -tune zerolatency \      # 저지연 모드 (B프레임 제거, 버퍼 최소화)
  -crf 23 \
  -g 60 -keyint_min 60 \   # 2초마다 I프레임 (HLS 세그먼트 경계)
  -sc_threshold 0 \        # 씬 컷 감지 비활성화 (일정한 GOP)
  -f hls -hls_time 2 \
  /hls/cam1/stream.m3u8

# H.265 → H.264 파일 변환 (저장본 재인코딩)
ffmpeg -i recording_h265.mp4 \
  -c:v libx264 -preset medium -crf 23 \
  -c:a copy \
  recording_h264.mp4
```

### CPU 폭증 문제

```
[소프트웨어 트랜스코딩 CPU 비용]
  스트림 복사 (-c copy):      CPU ~0.5%/채널
  H.264 → H.264 (copy):      CPU ~0.5%/채널
  H.265 → H.264 (libx264):   CPU ~80~200%/채널 ← 폭증
  4K H.265 → H.264:          CPU ~400%+/채널

[채널 수 × CPU]
  카메라 8대 × H.265 1080p → H.264:
    preset veryfast: ~8 × 80% = 640% (8코어 서버 포화)
    preset ultrafast: ~8 × 40% = 320%

[해결 전략]
  1. 하드웨어 가속 인코딩 (권장)
     NVIDIA NVENC, Intel QSV, VAAPI
     → CPU 사용 ~5%/채널으로 감소

  2. MediaMTX 트랜스코딩 (내장 FFmpeg 기반)

  3. 이중 스트림 전략
     카메라가 H.264 서브스트림 제공 → 서브스트림 직접 사용
     (Hikvision, Dahua 등 대부분의 IP 카메라는 듀얼 스트림)
     → 메인: H.265 고화질(NVR 저장) / 서브: H.264 저화질(웹 스트리밍)
```

### 하드웨어 가속 H.265 → H.264 변환

```bash
# NVIDIA NVENC (CUDA 디코딩 + 인코딩)
ffmpeg -hwaccel cuda -hwaccel_output_format cuda \
  -c:v hevc_cuvid \        # GPU H.265 디코딩
  -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -c:v h264_nvenc \        # GPU H.264 인코딩
  -preset p3 \             # NVENC preset (p1=빠름~p7=느림)
  -cq 23 \
  -b:v 0 \
  -tune:v hq \
  -f hls -hls_time 2 \
  /hls/cam1/stream.m3u8

# Intel QSV (내장 GPU)
ffmpeg -hwaccel qsv \
  -c:v hevc_qsv \          # QSV H.265 디코딩
  -i rtsp://cam1/stream1 \
  -c:v h264_qsv \          # QSV H.264 인코딩
  -global_quality 23 \
  -f hls -hls_time 2 \
  /hls/cam1/stream.m3u8

# VAAPI (Linux 범용)
ffmpeg \
  -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
  -hwaccel_output_format vaapi \
  -c:v hevc_vaapi \
  -i rtsp://cam1/stream1 \
  -vf 'hwupload' \
  -c:v h264_vaapi \
  -qp 23 \
  -f hls -hls_time 2 \
  /hls/cam1/stream.m3u8
```

---

## 10. 트랜스코딩 시 타임스탬프 어긋남 (Time Drift)

### 원인

```
[문제 발생 시나리오]
  1. 카메라 → RTSP (H.265) → FFmpeg 트랜스코딩 → HLS

  2. 디코딩 + 인코딩 처리 시간이 일정하지 않음
     → 입력 PTS(Presentation Timestamp) vs 출력 DTS 불일치
     → 고부하 시 인코딩 지연 → 세그먼트 경계 어긋남
     → HLS 플레이어: "버퍼링" 또는 갑작스러운 점프

  3. H.265 B프레임 참조 복잡성
     → 디코딩 순서(DTS)와 표시 순서(PTS) 차이 더 큼
     → H.264로 변환 시 이 차이가 타임라인에 전파

  4. 카메라 시계 불안정
     → RTSP 타임스탬프 자체가 드리프트 발생
```

### 타임스탬프 보정 옵션

```bash
# 기본 방법: 입력 타임스탬프를 버리고 FFmpeg가 재생성
ffmpeg -rtsp_transport tcp \
  -use_wallclock_as_timestamps 1 \   # 수신 시각을 PTS로 사용
  -i rtsp://cam1/stream1 \
  -c:v libx264 -preset veryfast \
  -vsync cfr \                        # CFR: 일정한 프레임레이트 강제 (드리프트 방지)
  -r 25 \                             # 출력 프레임레이트 고정 (입력 불규칙 대응)
  -async 1 \                          # 오디오/비디오 동기화
  -f hls -hls_time 2 \
  /hls/cam1/stream.m3u8

# 타임스탬프 완전 초기화 (기준점 리셋)
ffmpeg -rtsp_transport tcp \
  -i rtsp://cam1/stream1 \
  -c:v libx264 -preset veryfast \
  -vf setpts=PTS-STARTPTS \          # PTS를 0부터 재시작
  -af asetpts=PTS-STARTPTS \
  -f hls ...

# 세그먼트 타임스탬프 리셋 (HLS 분할 파일)
ffmpeg ... \
  -f segment -segment_time 3600 \
  -reset_timestamps 1 \              # 각 파일마다 0부터 시작 (필수!)
  -strftime 1 \
  "recordings/cam1/%Y%m%d_%H%M%S.mp4"
```

### 전략별 정리

```
[이중 스트림으로 CPU 절약 + Time Drift 회피]
  카메라 설정:
    메인 스트림: H.265, 4Mbps → NVR/TimescaleDB 저장 (스트림 복사)
    서브 스트림: H.264, 512kbps, 15fps → 웹 스트리밍 (스트림 복사)

  장점:
    트랜스코딩 없음 → CPU 폭증 없음
    타임스탬프 어긋남 없음
    저화질이지만 실시간성 완벽 유지

  MediaMTX 설정:
    cam1-main:
      source: rtsp://cam1/Streaming/Channels/101   # H.265 메인
    cam1-sub:
      source: rtsp://cam1/Streaming/Channels/102   # H.264 서브 → 웹용

[부득이하게 트랜스코딩이 필요한 경우]
  1. use_wallclock_as_timestamps 1 + vsync cfr
  2. 하드웨어 가속으로 지연 최소화
  3. HLS 세그먼트를 짧게 (2초) → 드리프트 누적 최소화
  4. 재접속 옵션으로 드리프트 리셋:
     -reconnect_at_eof 1 → 타임스탬프 자연 초기화
```

---

## 11. 관련
- [[Video-Fundamentals]] — H.264/H.265 코덱, YUV 색상 포맷 이론
- [[Streaming-Protocols]] — RTSP, HLS, WebRTC 프로토콜 비교
- [[MediaMTX]] — FFmpeg와 연동하는 미디어 서버
- [[Spring-CCTV]] — Spring 기반 JavaCV vs FFmpeg 멀티프로세스 처리
