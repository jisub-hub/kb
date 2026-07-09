---
tags:
  - cctv
  - video
  - streaming
  - moc
  - index
created: 2026-06-16
---

# CCTV MOC

> IP 카메라 시스템의 영상 압축·색상 처리·스트리밍 프로토콜·오픈소스 서버 스택. 코덱 원리부터 FFmpeg 파이프라인, MediaMTX RTSP/WebRTC 서버까지.

## 영상 기초
- [[Video-Fundamentals]] — H.264/H.265/AV1 코덱, GOP/I/P/B 프레임, YUV/NV12 색상 포맷, 비트레이트 설계

## 스트리밍 프로토콜
- [[Streaming-Protocols]] — RTSP, RTMP, HLS, WebRTC, SRT 비교 및 선택 기준

## 오픈소스 도구
- [[FFmpeg]] — 트랜스코딩·스트림 복사·필터그래프·하드웨어 가속 (NVENC/VAAPI/VideoToolbox)
- [[MediaMTX]] — RTSP/RTMP/HLS/WebRTC/SRT 범용 미디어 서버, 인증·녹화·API

## Spring 통합
- [[Spring-CCTV]] — JavaCV(DJL 통합 가능) vs FFmpeg 서브프로세스 다채널 처리 비교 (ONVIF·PTZ는 Streaming-Protocols 참고)

## 관련
- [[../../infra/database/TimescaleDB]] — 영상 이벤트 메타데이터 시계열 저장
- [[../iot/IoT-Architecture]] — IoT + CCTV 통합 아키텍처 참고
