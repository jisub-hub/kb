---
tags:
  - hardware
  - gpu
  - ai
  - moc
  - index
created: 2026-06-16
---

# GPU Workstation MOC

> AI 워크로드 중심 GPU·워크스테이션 분석. 스펙·대역폭·소프트웨어 생태계·실전 적합성 기준으로 평가.

## 디바이스
- [[DGX-Spark]] — NVIDIA GB10 Grace Blackwell, 1 PetaFLOP AI, 128GB LPDDR5X, CUDA 생태계
- [[Apple-M5-Max]] — Apple M5 Max, 128GB 통합 메모리, 고대역폭, macOS/MLX 생태계

## GPU 스펙 레퍼런스
- [[GPU-Specs]] — NVIDIA GeForce 4080~5090, A100/H100/H200/B200, RTX A4000/A6000, L40S, RTX Pro Blackwell 종합 스펙·비교
- [[AMD-GPU]] — AMD MI300X/MI300A, CDNA 3 아키텍처, ROCm 스택, NVIDIA 비교

## 추론 원리
- [[Apple-Silicon-Inference]] — Apple Silicon LLM 추론: GPU Core·NPU·대역폭의 역할 (decode=대역폭, core=prefill, NPU≈미사용)
- [[Single-Local-AI-Machine]] — 단일 로컬 AI 머신 선택: M5 Ultra vs DGX Spark (워크로드·대역폭·생태계·하이브리드)

## 핵심 비교 지표 — AI 관점

| 지표 | DGX Spark | M5 Max |
|---|---|---|
| AI 연산 (FP4/INT4) | **1 PetaFLOP** | ~40 TOPS (Neural Engine) |
| 메모리 대역폭 | 273 GB/s | **~600+ GB/s** |
| 통합 메모리 | 128GB LPDDR5X | 128GB LPDDR5X |
| CUDA 생태계 | ✅ 완전 지원 | ❌ |
| MLX / CoreML | ❌ | ✅ |
| 주요 OS | DGX OS (Ubuntu) | macOS |

## 관련
- [[../rack-mount-server/Rack-Mount-Server]] · [[../blade-server/Blade-Server]]
