---
tags:
  - hardware
  - edge-ai
  - embedded
  - moc
  - index
created: 2026-06-17
---

# Edge AI MOC

> 엣지·임베디드 AI 하드웨어. 저전력·현장 배포·온디바이스 추론 중심. 클라우드 없이 현장에서 추론하는 워크로드용.

## 디바이스
- [[Jetson]] — NVIDIA Jetson (Orin Nano / AGX Orin / AGX Thor), CUDA 엣지 모듈, 로보틱스·CCTV·온디바이스 LLM

## 핵심 관점
- 데이터센터 GPU와 동일한 CUDA/TensorRT 스택을 ~10~130W로 사용
- decode TPS는 여기서도 대역폭이 천장 (100~273 GB/s) → 대형 모델은 느림
- 통합 메모리로 적재는 가능, 가치는 "빠름"이 아니라 "현장·저전력·오프라인"

## 관련
- [[../gpu-workstation/_index|GPU Workstation]] — DGX Spark, M5 Max (데스크탑 저전력)
- [[../../ai/SLLM]] · [[../../ai/DJL]] · [[../../ai/LLM-System-Performance-Analysis]]
