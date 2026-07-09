---
tags:
  - cctv
  - video
  - codec
  - color-format
  - h264
  - h265
  - av1
created: 2026-06-16
---

# 영상 기초 — 코덱·색상 포맷·컨테이너

> [!summary] 한 줄 요약
> 코덱은 **압축 알고리즘**, 색상 포맷은 **픽셀 데이터 표현 방식**, 컨테이너는 **파일/스트림 포장 형식**이다. CCTV 시스템 설계 시 H.265(압축률)·YUV420(색상 효율)·RTSP(스트리밍) 조합이 핵심.

---

## 1. 영상 데이터의 기본 구조

```
[물리적 관점]
  1프레임 = 가로×세로 픽셀의 집합
  1080p30 = 1920×1080픽셀 × 30프레임/초
  Raw 비트레이트 = 1920 × 1080 × 3bytes × 30 = ~187 MB/s  ← 저장 불가능
  H.264 압축 후 = ~2~8 Mbps                                  ← 저장·전송 가능

[시간적 관점]
  I-Frame: 완전한 정지 이미지 (키프레임, 크기 큼)
  P-Frame: 이전 프레임 대비 변화만 저장 (Predictive)
  B-Frame: 앞·뒤 프레임 참조 (Bi-directional, 압축률 최고)

[GOP (Group of Pictures)]
  I──P──B──P──B──P──B──P──I──P──B──...
  └────── GOP 길이 (보통 30~60프레임) ──────┘
  → GOP 내 I프레임이 많을수록 랜덤 접근↑, 파일 크기↑
  → CCTV: 짧은 움직임 탐지 → 짧은 GOP / 저장 효율 → 긴 GOP
```

---

## 2. 코덱 비교

### H.264 (AVC — Advanced Video Coding)

```
표준: ITU-T H.264 / MPEG-4 Part 10
출시: 2003년
Profile: Baseline / Main / High
Level: 4.0(1080p30), 4.1(1080p30 Hi-BR), 5.1(4K)

압축 원리:
  - 8×8 또는 4×4 블록 DCT (Discrete Cosine Transform)
  - 인트라 예측 (같은 프레임 내 이웃 블록 참조)
  - 인터 예측 (이전/이후 프레임에서 움직임 벡터 탐색)
  - CABAC 엔트로피 코딩 (High Profile, 최고 압축)

비트레이트 (1080p30):
  스트리밍: 3~5 Mbps
  CCTV 저장: 2~4 Mbps (정적 장면 多)
  고품질: 8~10 Mbps

강점: 하드웨어 가속 지원 광범위, 호환성 최고
약점: H.265 대비 2배 비트레이트 (같은 품질 기준)
```

### H.265 (HEVC — High Efficiency Video Coding)

```
표준: ITU-T H.265 / MPEG-H Part 2
출시: 2013년
Profile: Main / Main 10 / Main Still Picture

H.264 대비:
  동일 화질 기준 비트레이트 50% 절감
  CTU (Coding Tree Unit): 최대 64×64 블록 (H.264: 16×16 MB)
  향상된 인트라 예측 (35방향 vs H.264 9방향)
  병렬 처리: Tile, Slice → 멀티코어 활용

비트레이트 (1080p30):
  CCTV 저장: 1~2 Mbps (H.264 대비 절반)
  4K CCTV: 4~8 Mbps (H.264: 8~16 Mbps)

강점: CCTV 저장 비용 절감 핵심, 4K 표준
약점: 인코딩 CPU 부하 2~3배↑, 일부 기기 디코딩 미지원
     특허 문제 (HEVC Advance, MPEG LA 이중 라이선스)
```

### H.266 (VVC — Versatile Video Coding)

```
출시: 2020년
H.265 대비 50% 추가 압축 (H.264 대비 75% 절감)
아직 하드웨어 가속 지원 초기 단계
8K, VR 콘텐츠 타겟
```

### AV1

```
개발: Alliance for Open Media (Google, Meta, Apple, Netflix 등)
출시: 2018년 (로열티 무료)

H.265 대비:
  30~50% 추가 압축 효율
  CDEF, LR (Loop Restoration Filter) 등 혁신적 필터
  소프트웨어 인코딩: H.265보다 5~20배 느림

하드웨어 지원:
  인텔 12세대+, NVIDIA RTX 30+, AMD RDNA 2+
  모바일: Apple A17 Pro+, Snapdragon 8 Gen 2+

강점: 완전 오픈소스 로열티 무료, 웹 스트리밍 미래 표준
약점: 인코딩 복잡도 높음, CCTV 카메라 지원 미미
```

### 코덱 선택 가이드

| 용도 | 권장 코덱 | 이유 |
|------|----------|------|
| CCTV 저장 (IP 카메라) | **H.265** | 저장 비용 50% 절감 |
| 레거시 장비 연동 | H.264 | 호환성 최고 |
| 웹 브라우저 스트리밍 | H.264 또는 AV1 | 브라우저 기본 지원 |
| 4K/8K 방송 | H.265 또는 H.266 | 대역폭 효율 |
| OTT 스트리밍 | AV1 | 로열티 없음, Netflix/YouTube |

---

## 3. 색상 포맷 (Chroma Subsampling)

인간 눈은 **밝기(Luma)에 민감, 색상(Chroma)에 덜 민감** → 색상 정보를 줄여도 체감 품질 유지.

### YUV / YCbCr 구조

```
Y  = Luma (밝기)
Cb = 파란색 색차 (Blue difference)
Cr = 빨간색 색차 (Red difference)

YUV 4:4:4 — 서브샘플링 없음
  Y: 4픽셀, Cb: 4픽셀, Cr: 4픽셀
  원본 색상 완전 보존, 크기 100%

YUV 4:2:2 — 수평 2:1 서브샘플링
  Y: 4픽셀, Cb: 2픽셀, Cr: 2픽셀
  방송 표준, 크기 75%

YUV 4:2:0 — 수평·수직 2:1 서브샘플링 ⭐
  Y: 4픽셀, Cb: 1픽셀, Cr: 1픽셀
  H.264/H.265 기본값, 크기 50%
  CCTV, 스트리밍 표준

YUV 4:0:0 — 색상 없음 (그레이스케일)
  Y만 존재
  흑백 카메라, 야간 모드
```

### 메모리 레이아웃 (YUV 4:2:0 변형)

```
[I420 / YU12] — Planar (Y, U, V 별도 평면)
  YYYYYYYYYYY...  (Y 전체)
  UUUUU...        (U = Cb, 1/4 크기)
  VVVVV...        (V = Cr, 1/4 크기)
  FFmpeg: -pix_fmt yuv420p

[NV12] — Semi-Planar (Y 평면 + UV 인터리브)
  YYYYYYYYYYY...  (Y 전체)
  UVUVUVUV...     (U,V 교번, 1/2 크기)
  하드웨어 디코더 선호 (NVIDIA, 인텔 QuickSync)
  FFmpeg: -pix_fmt nv12

[NV21] — NV12와 UV 순서 반전
  Android 카메라 기본 포맷

[YUY2 / YUYV] — Packed (Y,U,Y,V 인터리브, 4:2:2)
  웹캠(V4L2) 출력 기본값
```

### RGB 포맷

```
RGB24 / BGR24 — 8비트 × 3채널 = 24bpp
  OpenCV 기본: BGR 순서 (주의!)
  완전한 색상, 압축 없음

RGBA32 / BGRA32 — 알파 채널 포함, 32bpp

RGB565 — 5+6+5비트, 16bpp
  임베디드 디스플레이 (메모리 절약)
```

---

## 4. 컨테이너 포맷

컨테이너는 비디오·오디오·메타데이터·자막을 하나의 파일/스트림으로 포장한다.

| 컨테이너 | 확장자 | 스트리밍 지원 | 특징 |
|---------|--------|------------|------|
| **MP4** | .mp4 | 제한적 (fragmented MP4) | 범용, H.264/H.265/AV1 |
| **MKV** | .mkv | ❌ | 오픈소스, 다중 트랙, CCTV 저장 |
| **TS** | .ts | ✅ | MPEG-2 Transport Stream, HLS |
| **FLV** | .flv | ✅ | 레거시 RTMP, 현재 비권장 |
| **WebM** | .webm | ✅ | VP8/VP9/AV1, 브라우저 네이티브 |
| **MOV** | .mov | ❌ | Apple QuickTime 기반 |

### CCTV 저장 권장

```
NVR/DVR 저장: H.265 + MP4 (fragmented) 또는 MKV
  → 파일 분할: 1시간 또는 1GB 단위
  → 인덱싱: 시간 기반 디렉토리 구조
    /recordings/2026/06/16/14/cam01_14-00-00.mp4

HLS 스트리밍: H.264 + TS 세그먼트
  → 5초 세그먼트, m3u8 플레이리스트
  → 웹 브라우저 실시간 모니터링

RTSP: H.264/H.265 + RTP
  → 낮은 지연 (< 1초)
  → VMS(Video Management Software) 연동
```

---

## 5. 비트레이트 설계

```
저장 용량 계산:
  비트레이트(Mbps) × 3600초/시간 ÷ 8 = MB/시간
  예: 4 Mbps × 3600 ÷ 8 = 1,800 MB/시간 = 1.8 GB/시간

카메라 30대 × 4 Mbps × 24시간 × 30일 저장:
  = 30 × 1.8 GB × 24 × 30 = 38,880 GB ≈ 39 TB

H.265 사용 시 (동일 화질):
  = 39 TB × 0.5 ≈ 20 TB → 저장 비용 절반
```

### CBR vs VBR

| | CBR (Constant Bitrate) | VBR (Variable Bitrate) |
|--|----------------------|----------------------|
| 비트레이트 | 고정 | 장면 복잡도에 따라 가변 |
| 저장 예측 | 쉬움 | 어려움 |
| 화질 | 정적 장면 낭비 | 동적 장면 적응 |
| CCTV 권장 | 네트워크 대역폭 제한 시 | 저장 효율 우선 시 |

---

## 6. 관련
- [[FFmpeg]] — 코덱·포맷 변환, 트랜스코딩, 스트리밍 파이프라인
- [[MediaMTX]] — RTSP/RTMP/HLS/WebRTC 스트리밍 서버
- [[Streaming-Protocols]] — RTSP, RTMP, HLS, WebRTC, SRT 비교
