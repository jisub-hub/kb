---
tags:
  - hardware
  - jetson
  - edge-ai
  - embedded
  - nvidia
  - inference
created: 2026-06-17
---

# NVIDIA Jetson — 엣지 AI 임베디드 플랫폼

> [!summary] 한 줄 요약
> Jetson은 **CUDA를 그대로 쓰는 손바닥 크기 엣지 AI 모듈**이다. 로보틱스·자율주행·CCTV·온디바이스 LLM처럼 클라우드 없이 현장에서 추론해야 하는 워크로드를 ~10~130W 전력으로 처리한다. 데이터센터 GPU와 같은 CUDA/TensorRT 스택을 공유하는 게 핵심 강점.

---

## 1. Jetson이란 — 데이터센터 GPU와 무엇이 다른가

```
H100/B200 (데이터센터)   →  랙, 수백~수천 W, HBM 3~8 TB/s, 클라우드 서빙
DGX Spark (데스크탑)      →  개인 워크스테이션, 170W, 273 GB/s
Jetson (엣지/임베디드)    →  손바닥 크기, 10~130W, 100~273 GB/s, 현장 배포
```

Jetson은 **SoC(System-on-Chip)** — Arm CPU + NVIDIA GPU + 메모리 컨트롤러 + 비디오 인코더/디코더 + DLA(Deep Learning Accelerator)를 한 칩에 통합한다. 데스크탑 GPU와 달리:

- **통합 메모리(Unified Memory)** — CPU·GPU가 LPDDR5 메모리 풀을 공유 (Apple Silicon·DGX Spark와 유사 구조)
- **저전력** — 전력 모드(`nvpmodel`)로 10W~130W 조절
- **CUDA/TensorRT 완전 호환** — 데이터센터에서 만든 모델을 거의 그대로 배포
- **I/O 풍부** — CSI 카메라, GPIO, CAN, PCIe, 다수 USB → 로보틱스/비전 직결

---

## 2. 라인업 (2026 기준)

| 모듈 | AI 성능 | 메모리 | 대역폭 | 전력 | 용도 |
|------|--------|--------|--------|------|------|
| Orin Nano (Super) | 67 TOPS (INT8) | 8 GB LPDDR5 | 102 GB/s | 7~25W | 입문·소형 비전·SLM |
| Orin NX | 70~157 TOPS | 8/16 GB | 102 GB/s | 10~40W | 드론·소형 로봇 |
| **AGX Orin 64GB** | **275 TOPS** | 64 GB LPDDR5 | **204.8 GB/s** | 15~60W | 자율주행·멀티카메라·LLM |
| **AGX Thor** | ~2,070 FP4 TFLOPS (sparse) | 128 GB LPDDR5X | **273 GB/s** | ~130W | 휴머노이드·차세대 로보틱스·온디바이스 LLM |

> **AGX Thor**(2025, Blackwell 세대)는 Jetson 최상위. Blackwell GPU + FP4 지원으로 엣지에서 대형 모델·VLM·멀티모달을 노린다. AGX Orin은 Ampere 세대로 여전히 주력 양산 플랫폼.

---

## 3. LLM 추론 관점 — v3 보고서 논리 적용

[[LLM-System-Performance-Analysis|v3 시스템 보고서]]의 핵심 명제(decode TPS ≈ 대역폭 / 모델크기)가 Jetson에도 그대로 적용된다. Jetson은 **대역폭이 낮은 게 천장**이다.

```
decode TPS ≈ B_HBM / W   (단일 사용자, memory-bound)

AGX Orin 64GB (204.8 GB/s):
  7B  Q4 (~4GB):   204.8 / 4  ≈ 51 TPS
  13B Q4 (~7GB):   204.8 / 7  ≈ 29 TPS
  34B Q4 (~18GB):  204.8 / 18 ≈ 11 TPS
  70B Q4 (~35GB):  204.8 / 35 ≈ 5.8 TPS  (적재는 되나 느림)

AGX Thor 128GB (273 GB/s):
  70B Q4 (~35GB):  273 / 35   ≈ 7.8 TPS
  + FP4 연산으로 prefill·VLM에서 Orin 대비 큰 우위
```

> [!note] 적재 가능성 vs 속도 — v3 5절과 동일한 함정
> AGX Orin 64GB / Thor 128GB는 통합 메모리가 커서 **대형 모델 적재는 가능**하다. 하지만 대역폭(204~273 GB/s)이 데이터센터 GPU의 1/15~1/30이라 **TPS는 낮다.**
> Jetson은 "큰 모델을 빠르게"가 아니라 "**현장에서·전력 제약 하에서·네트워크 없이**" 돌리는 게 본질이다.

**실전 권장 모델 크기:**
- Orin Nano 8GB → 3B~7B 양자화 (Phi, Gemma 2B/9B, Llama 3.2 3B) → [[SLLM]] 참고
- AGX Orin 64GB → 13B~34B 양자화, 멀티모달 [[VLM]]
- AGX Thor 128GB → 70B Q4까지, 온디바이스 에이전트

---

## 4. 소프트웨어 스택

```
JetPack SDK              # Jetson 전용 OS + 라이브러리 번들
├─ L4T (Linux for Tegra)  # Ubuntu 기반
├─ CUDA / cuDNN           # 데이터센터와 동일 API
├─ TensorRT              # 추론 최적화 (FP16/INT8/FP4 양자화)
├─ DeepStream            # 멀티스트림 비디오 분석 파이프라인
└─ Triton Inference Server # 엣지 모델 서빙

LLM 런타임:
  llama.cpp (CUDA backend)  # GGUF Q4_K_M 등
  ollama                    # Jetson 지원
  MLC-LLM / TensorRT-LLM    # 최적화 배포
```

TensorRT가 핵심이다 — [[../../ai/Inference-Optimization|추론 최적화]]의 fused kernel·INT8/FP4 양자화를 엣지에서 그대로 적용해 제한된 대역폭을 최대한 짜낸다.

---

## 5. 어디에 쓰는가

| 도메인 | 활용 |
|--------|------|
| 로보틱스 | 자율 이동 로봇(AMR), 매니퓰레이터, 휴머노이드 (Thor) |
| 자율주행 | ADAS, 멀티카메라 인식, 센서 퓨전 |
| 스마트 비전 | [[../../ai/DJL|CCTV·YOLO 객체 감지]], 결함 검사, 리테일 분석 |
| 온디바이스 LLM | 오프라인 음성 비서, 프라이버시 민감 환경, 네트워크 단절 현장 |
| 산업 IoT | 엣지 예지보전, 실시간 이상 탐지 |

> [[../../ai/DJL]]의 CCTV YOLO 파이프라인을 Jetson + DeepStream으로 옮기면 클라우드 전송 없이 현장에서 다채널 추론이 가능하다.

---

## 6. Jetson vs 다른 엣지/저전력 옵션

| | Jetson AGX Thor | DGX Spark | Apple M5 Max | RTX 4090 |
|--|----------------|-----------|--------------|----------|
| 폼팩터 | 임베디드 모듈 | 데스크탑 | 노트북/데스크탑 | 데스크탑 PCIe |
| 대역폭 | 273 GB/s | 273 GB/s | ~600 GB/s | 1.0 TB/s |
| 통합 메모리 | 128 GB | 128 GB | 128 GB | 24 GB (전용) |
| 전력 | ~130W | ~170W | ~92W | ~450W |
| CUDA | ✅ | ✅ | ❌ (MLX) | ✅ |
| 현장 배포(I/O, 내구성) | ✅✅ | ❌ | ❌ | ❌ |
| 용도 | 엣지·로보틱스 | 개인 개발 | 로컬 추론 | 워크스테이션 |

> DGX Spark와 Thor는 대역폭(273 GB/s)·메모리(128GB)가 비슷하지만, **Thor는 카메라·GPIO·CAN 등 현장 I/O와 내구성**을 갖춘 임베디드 모듈이고 DGX Spark는 책상 위 개발 장비다. 같은 칩 계열이라도 타겟이 다르다.

---

## 7. 핵심 정리

```
1. Jetson = CUDA를 쓰는 저전력 엣지 AI 모듈 (데이터센터 스택 그대로)
2. decode TPS는 여기서도 대역폭이 천장 — 100~273 GB/s라 대형 모델은 느림
3. 통합 메모리로 적재는 되지만, "빠름"이 아니라 "현장·저전력·오프라인"이 가치
4. SLM/양자화 모델 + TensorRT + DeepStream 조합이 정석
5. 로보틱스·CCTV·자율주행·온디바이스 LLM에 최적
```

---

## 관련
- [[DGX-Spark]] — 같은 GB10/Blackwell 계열 데스크탑 버전
- [[Apple-M5-Max]] — 통합 메모리 저전력 추론 비교
- [[../../ai/LLM-System-Performance-Analysis]] — 대역폭/모델크기 TPS 공식 (v3)
- [[../../ai/SLLM]] — 엣지에 적합한 소형 모델
- [[../../ai/DJL]] — CCTV YOLO 객체 감지 (Jetson 배포 후보)
- [[../../ai/Inference-Optimization]] — TensorRT 양자화·최적화
