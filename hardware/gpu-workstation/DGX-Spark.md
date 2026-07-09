---
tags:
  - hardware
  - nvidia
  - dgx
  - ai
  - blackwell
  - cuda
created: 2026-06-16
---

# NVIDIA DGX Spark

> [!summary] 한 줄 요약
> NVIDIA가 2025년 GTC에서 공개한 **데스크탑 폼팩터 개인용 AI 슈퍼컴퓨터**. GB10 Grace Blackwell SoC를 탑재해 1 PetaFLOP AI 연산을 제공하며, 클라우드 GPU 클러스터 없이 로컬에서 LLM 추론·파인튜닝이 가능하다.

---

## 1. SoC — GB10 Grace Blackwell Superchip

DGX Spark의 핵심은 **하나의 칩에 Grace CPU와 Blackwell GPU를 통합**한 GB10 SoC.

```
┌───────────────────────────────────────────────────────────┐
│               GB10 Grace Blackwell Superchip               │
│                                                             │
│  ┌──────────────────────┐      ┌──────────────────────┐   │
│  │   Grace CPU          │      │   Blackwell GPU       │   │
│  │   (ARM Neoverse N2)  │      │   (B10 다이)          │   │
│  │   20코어             │◄────►│   1 PetaFLOP (FP4)   │   │
│  │   LPDDR5X 컨트롤러   │NVLink│   Tensor Core 4세대   │   │
│  └──────────────────────┘ C2C └──────────────────────┘   │
│           │                                │               │
│           └────────────┬───────────────────┘               │
│                        │ 128GB LPDDR5X                     │
│                        │ 통합 메모리 풀                     │
└───────────────────────────────────────────────────────────┘
```

**NVLink-C2C (Chip-to-Chip)**: CPU와 GPU 간 인터커넥트. 900 GB/s 양방향 대역폭으로 별도 PCIe 병목 없이 통합 메모리 풀에 접근.

---

## 2. 전체 사양 (128GB / 4TB 구성)

### CPU — NVIDIA Grace (ARM Neoverse N2)

| 항목 | 사양 |
|---|---|
| 아키텍처 | ARM Neoverse N2 (ARMv9) |
| 코어 수 | 20코어 |
| 캐시 | L3 캐시 40MB |
| 명령어셋 | ARMv9-A, SVE2, AMX |
| 공정 | TSMC 4nm |
| 특징 | Confidential Computing, PCIe Gen5 호스트 |

### GPU — Blackwell (B10)

| 항목 | 사양 |
|---|---|
| 아키텍처 | Blackwell (5세대 Tensor Core) |
| AI 성능 | **1 PetaFLOP** (FP4 Sparsity) |
| FP8 Dense | ~500 TFLOPS |
| FP16/BF16 | ~250 TFLOPS |
| FP32 | ~125 TFLOPS |
| Transformer Engine | ✅ 4세대 (FP8 자동 혼합 정밀도) |
| 공정 | TSMC 4nm |

### 메모리

| 항목 | 사양 |
|---|---|
| 용량 | **128GB** |
| 타입 | LPDDR5X |
| 구성 | CPU·GPU 통합 통일 메모리 풀 |
| 대역폭 | **273 GB/s** |
| ECC | ✅ 지원 |
| NVLink-C2C | 900 GB/s (CPU↔GPU 인터커넥트) |

> [!important] 273 GB/s의 의미
> 클라우드급 H100 SXM의 HBM3는 3.35 TB/s. DGX Spark는 LPDDR5X 기반이라 대역폭이 낮다.
> 반면 **128GB 전체가 GPU VRAM으로 사용 가능** → 70B 파라미터 LLM을 통째로 올릴 수 있다.
> 대역폭보다 **메모리 용량**이 LLM 추론의 실질적 병목일 때 강점.

### 스토리지

| 항목 | 사양 |
|---|---|
| 용량 | **4TB** |
| 타입 | NVMe SSD (PCIe Gen5) |
| 읽기 속도 | ~14 GB/s (PCIe Gen5 NVMe) |
| 쓰기 속도 | ~10 GB/s |
| 용도 | 모델 가중치 저장, 데이터셋, 체크포인트 |

### 네트워크

| 항목 | 사양 |
|---|---|
| NIC | NVIDIA ConnectX-7 내장 |
| 이더넷 | 100GbE (QSFP28) |
| InfiniBand | 400Gb/s HDR (선택) |
| 용도 | 다중 DGX Spark 클러스터링 (DGX-to-DGX NVLink Bridge) |

> DGX Spark 2대를 NVLink Bridge로 연결하면 **256GB 통합 메모리 풀**로 확장 가능.

### 기타

| 항목 | 사양 |
|---|---|
| 폼팩터 | 데스크탑 미니 PC |
| TDP | ~170W |
| 전원 | 단일 외부 어댑터 |
| 크기 | 약 27 × 27 × 7 cm |

---

## 3. 운영체제

### DGX OS (기본 탑재)

```
기반: Ubuntu 22.04 LTS
커널: NVIDIA 커스텀 패치 커널
사전 설치:
  - CUDA Toolkit (최신)
  - cuDNN, cuBLAS, NCCL
  - NVIDIA Container Toolkit (Docker/K8s)
  - TensorRT
  - NVIDIA NIM (Neural Inference Microservice)
  - NVIDIA AI Enterprise 라이선스 포함
```

### 지원 OS

| OS | 지원 수준 |
|---|---|
| **DGX OS (Ubuntu 22.04)** | ✅ 완전 지원, 사전 최적화 |
| Ubuntu 22.04 / 24.04 | ✅ CUDA 드라이버 설치 후 |
| RHEL 9 / Rocky Linux 9 | ✅ 공식 지원 |
| Windows | ❌ |
| macOS | ❌ |

---

## 4. CUDA 소프트웨어 스택

DGX Spark의 가장 큰 강점은 **완전한 CUDA 생태계**.

```
애플리케이션 레이어
  LLaMA.cpp · vLLM · HuggingFace Transformers · JAX · PyTorch · TensorFlow

프레임워크 최적화 레이어
  NVIDIA NIM (추론 마이크로서비스)
  TensorRT-LLM (LLM 추론 최적화, FP8/FP4 양자화)
  TensorRT (일반 추론 최적화)

CUDA 런타임 레이어
  CUDA 12.x
  cuDNN 9.x (딥러닝 프리미티브)
  cuBLAS (선형 대수)
  NCCL (통신)
  Triton Inference Server

드라이버 레이어
  NVIDIA GPU 드라이버 (Blackwell 지원)
  NVLink-C2C 드라이버
```

### LLM 추론 성능 (참고 수치)

| 모델 | 메모리 요구량 | 추론 가능 여부 | 주요 방식 |
|---|---|---|---|
| Llama 3.1 8B (FP16) | ~16GB | ✅ 여유 | 풀 정밀도 |
| Llama 3.1 70B (FP16) | ~140GB | ✅ 128GB 내 FP8 | FP8/INT8 양자화 |
| Llama 3.1 70B (INT4) | ~35GB | ✅ | GPTQ/AWQ |
| Llama 3.1 405B (INT4) | ~200GB | ❌ 단독 불가 | NVLink Bridge 2대 필요 |
| Mistral 7B | ~14GB | ✅ | 풀 정밀도 |

---

## 5. 파인튜닝 지원

```python
# HuggingFace + PEFT (LoRA) 예시
from transformers import AutoModelForCausalLM, TrainingArguments
from peft import LoraConfig, get_peft_model

# DGX Spark: 128GB VRAM → 70B 모델 LoRA 파인튜닝 가능
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70B",
    torch_dtype=torch.bfloat16,
    device_map="cuda"            # GB10 GPU에 자동 할당
)

config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"])
model = get_peft_model(model)
```

```bash
# NVIDIA NIM으로 즉시 추론 서버 배포
docker run --gpus all \
  -e NVIDIA_API_KEY=$API_KEY \
  -p 8000:8000 \
  nvcr.io/nim/meta/llama-3.1-70b-instruct:latest
```

---

## 6. 클러스터링 — DGX-to-DGX NVLink Bridge

```
[DGX Spark #1] ─── NVLink Bridge ─── [DGX Spark #2]
  128GB VRAM                              128GB VRAM
         ↓
   256GB 통합 VRAM 풀
   Llama 3.1 405B INT4 추론 가능
```

- NVLink Bridge: 두 대를 물리적으로 직결
- 논리적으로 하나의 GPU처럼 동작
- 대역폭: NVLink 4세대 기준 ~900 GB/s 양방향

---

## 7. 장점 / 단점

### ✅ 장점
- **완전한 CUDA 생태계** — PyTorch, JAX, TensorRT-LLM, vLLM 등 모든 GPU 프레임워크
- **1 PetaFLOP AI** — 데스크탑 폼팩터 대비 압도적 AI 연산 성능
- **128GB 통합 VRAM** — 70B 파라미터 모델 로컬 추론 가능
- **NIM 사전 지원** — NVIDIA 최적화 추론 컨테이너 즉시 배포
- **클러스터링** — NVLink Bridge로 2대 연결, 256GB 풀 구성
- **ECC 메모리** — 장시간 학습 작업에서 데이터 무결성

### ❌ 단점
- **메모리 대역폭 273 GB/s** — Apple M5 Max의 절반 수준 → 배치 추론 처리량 제한
- **ARM 기반** — x86 네이티브 소프트웨어 호환성 제한 (일부 도구 ARM 빌드 필요)
- **Linux 전용** — Windows/macOS 사용 불가
- **가격** — $3,000~$5,000 수준 (출시 당시)
- **LPDDR5X 공유 메모리** — CPU·GPU 동시 사용 시 대역폭 경쟁

---

## 8. 관련
- [[Apple-M5-Max]] · [[Single-Local-AI-Machine]] (단일 장비 선택 비교) · [[_index]]
