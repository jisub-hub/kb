---
tags:
  - hardware
  - apple
  - m5
  - ai
  - unified-memory
  - mlx
created: 2026-06-16
---

# Apple M5 Max

> [!summary] 한 줄 요약
> Apple Silicon의 최상위 칩. **단일 다이에 CPU·GPU·Neural Engine·메모리 컨트롤러를 통합**하고, LPDDR5X 기반 ~600+ GB/s 메모리 대역폭으로 LLM 배치 추론에서 경쟁력을 가진다. macOS 생태계와 MLX 프레임워크로 Apple 특화 AI 워크로드에 최적화.

---

## 1. 아키텍처 — Apple Unified Memory Architecture (UMA)

```
┌───────────────────────────────────────────────────────────────────┐
│                      M5 Max SoC (Package)                          │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                   Unified Memory (128GB LPDDR5X)              │  │
│  │                      ~600+ GB/s                               │  │
│  └───────┬──────────────────────┬───────────────────────────────┘  │
│          │                      │                                   │
│  ┌───────┴────────┐   ┌─────────┴────────┐   ┌──────────────────┐ │
│  │   CPU          │   │   GPU             │   │  Neural Engine   │ │
│  │  16코어         │   │  40코어           │   │  16코어          │ │
│  │  (12P + 4E)    │   │  ~40 TFLOPS(FP16)│   │  ~40+ TOPS       │ │
│  └────────────────┘   └──────────────────┘   └──────────────────┘ │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  미디어 엔진 · Secure Enclave · ISP · Thunderbolt 5 · PCIe  │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

**UMA(Unified Memory Architecture)의 핵심**: CPU·GPU·Neural Engine이 **동일한 물리 메모리 풀을 공유**. 데이터 복사 없이 각 연산 유닛이 동일 메모리를 참조 → 전통적인 discrete GPU 시스템 대비 메모리 복사 오버헤드 제거.

---

## 2. 전체 사양 (128GB / 4TB 구성)

### CPU

| 항목 | 사양 |
|---|---|
| 아키텍처 | Apple Silicon (ARMv8.6-A, 커스텀 마이크로아키텍처) |
| 코어 구성 | **16코어** (12 Performance + 4 Efficiency) |
| Performance 코어 | "Everest" 기반, 단일 스레드 성능 최상위급 |
| Efficiency 코어 | "Sawtooth" 기반, 저전력 백그라운드 작업 |
| L1 캐시 (P-core) | 192KB I + 128KB D |
| L2 캐시 | P클러스터 공유 32MB + E클러스터 4MB |
| L3 캐시 (SLC) | 48MB (System Level Cache) |
| 공정 | TSMC 3nm 2세대 (N3E) |
| ISA | ARMv8.6-A + Apple 확장 (AMX2, SME) |

### GPU

| 항목 | 사양 |
|---|---|
| 코어 수 | **40코어** Apple GPU (최대 구성) |
| 성능 | ~40+ TFLOPS (FP16) |
| API | Metal 3, MetalFX (업스케일링) |
| 특징 | Tile-Based Deferred Rendering, MPS (Metal Performance Shaders) |
| AI 가속 | Metal MPS를 통한 행렬 연산 가속 |

> [!note] GPU TFLOPS와 AI 성능
> Apple GPU의 40코어는 NVIDIA 방식의 TFLOPS 수치가 공식 발표되지 않는다.
> M4 Max (40코어) 실측 ~38 TFLOPS 기준, M5 Max (40코어)는 아키텍처 개선으로 ~40+ TFLOPS 추정.
> 이 수치는 Neural Engine의 AI 연산과 별도이며, **LLM은 주로 GPU MPS가 담당**.

### Neural Engine

| 항목 | 사양 |
|---|---|
| 코어 수 | 16코어 Neural Engine |
| 성능 | **~40+ TOPS** (정수 연산) |
| 용도 | CoreML 모델, Face ID, 사진 처리, On-Device 분류 |
| 특징 | INT8/INT4 특화, 소형 분류·회귀 모델 최적 |

> Neural Engine은 행렬 곱이 중심인 LLM 추론에는 직접 활용되지 않는다.
> GPU 코어·NPU·대역폭이 추론에서 각각 무엇을 담당하는지(decode=대역폭, core=prefill) → [[Apple-Silicon-Inference]]
> LLM 추론은 GPU(Metal MPS 또는 MLX)가 담당.

### 메모리 — 핵심 차별점

| 항목 | 사양 |
|---|---|
| 용량 | **128GB** |
| 타입 | LPDDR5X |
| 대역폭 | **~600+ GB/s** (M4 Max 실측: 546 GB/s 기준 M5 Max 예상치) |
| 구성 | CPU·GPU·Neural Engine 통합 통일 메모리 |
| ECC | ✅ 지원 |
| 접근 방식 | zero-copy (CPU↔GPU 복사 없음) |

> [!important] 600 GB/s의 의미 — DGX Spark 대비 분석
> 
> | | DGX Spark | M5 Max |
> |---|---|---|
> | 메모리 대역폭 | 273 GB/s | ~600+ GB/s |
> | AI FLOPS | 1 PetaFLOP (FP4) | ~45 TFLOPS (FP16) |
> | 배치 추론 처리량 | 낮음 (대역폭 제한) | **높음** (대역폭 여유) |
> | 대형 모델 로딩 | 가능 (VRAM 128GB) | 가능 (동일 128GB) |
> 
> **LLM 추론의 병목**:  
> - 소형 모델(배치 = 1): 메모리 대역폭이 병목 → M5 Max 유리  
> - 대규모 배치 추론: 연산 FLOPS가 병목 → DGX Spark 유리  
> - 실용 결론: 개발/테스트 추론은 M5 Max, 프로덕션 대용량 처리는 DGX Spark

### 스토리지

| 항목 | 사양 |
|---|---|
| 용량 | **4TB** |
| 타입 | Apple 커스텀 NVMe (PCIe Gen4 기반) |
| 읽기 속도 | ~8 GB/s |
| 쓰기 속도 | ~7 GB/s |
| 특징 | Apple T2 암호화 엔진 통합, APFS 최적화 |

> Apple NVMe는 공개 PCIe Gen5보다 읽기 속도가 낮지만, **Random 4K IOPS**는 경쟁력 있음.
> 모델 가중치 로딩 시 순차 읽기가 지배적이므로 DGX Spark PCIe Gen5 NVMe 대비 2배 느림.

### 기타

| 항목 | 사양 |
|---|---|
| 폼팩터 | MacBook Pro 16" / Mac Studio / Mac Pro (탑재 기기별 상이) |
| TDP | ~92W (Max 구성 풀로드 시) |
| Thunderbolt 5 | 120Gbps, 최대 8K 외부 디스플레이 |
| USB 4 | ✅ |
| Wi-Fi | Wi-Fi 7 (802.11be) |
| Bluetooth | Bluetooth 5.3 |

---

## 3. 운영체제

### macOS (기본·최적화 환경)

```
기반: macOS 16.x (Tahoe 예상) / 현재 macOS 15.x Sequoia
최적화 레이어:
  - Apple Neural Engine 드라이버 → CoreML
  - Metal / Metal Performance Shaders (MPS)
  - MLX (Apple's ML Framework for Apple Silicon)
  - Accelerate Framework (BLAS/LAPACK Apple 최적화)
  - Grand Central Dispatch (비동기 CPU 작업 스케줄링)
```

### 지원 OS 및 제약

| OS | 지원 수준 | 비고 |
|---|---|---|
| **macOS** | ✅ 완전 지원, 최대 성능 | Metal, CoreML, MLX 모두 활용 |
| **Asahi Linux** | ⚠️ 부분 지원 | GPU 드라이버 미완성 (Metal 사용 불가) |
| **UTM / Parallels (VM)** | ⚠️ 가상화 | ARM Linux 가능, GPU 직접 접근 제한 |
| **Docker (ARM)** | ✅ ARM 이미지 지원 | CUDA 사용 불가 |
| **Windows** | ❌ (Parallels로 ARM Windows 가능) | CUDA 불가 |
| **CUDA** | ❌ | NVIDIA 독점 → Metal/MPS 대체 필요 |

> [!warning] CUDA 부재
> AI 생태계 대부분이 CUDA 기반. PyTorch는 `device="mps"`로 M5 Max GPU 활용 가능하지만,
> cuDNN·TensorRT·vLLM 등 CUDA 최적화 라이브러리는 **직접 사용 불가**.
> MLX 또는 llama.cpp Metal 백엔드로 대체.

---

## 4. AI 소프트웨어 스택

```
애플리케이션 레이어
  Ollama · LM Studio · llama.cpp (Metal) · mlx-lm

프레임워크 레이어
  MLX (Apple) — Apple Silicon 특화 ML 프레임워크 (NumPy-like API)
  PyTorch (MPS 백엔드) — device="mps"로 GPU 가속
  JAX (metal plugin) — 실험적

런타임 최적화 레이어
  CoreML — 변환된 모델 (.mlpackage) 최적 추론 (Neural Engine + GPU 혼합)
  Metal Performance Shaders (MPS) — GPU 행렬 연산
  Accelerate — CPU BLAS/LAPACK 최적화

드라이버 레이어
  Metal API (Apple GPU 드라이버)
  CoreML 런타임
```

### MLX — Apple Silicon 특화 ML 프레임워크

Apple이 직접 관리하는 오픈소스 ML 프레임워크. NumPy 유사 API로 Metal GPU를 네이티브 가속.  
**mlx-lm**은 MLX 위에서 LLM 추론·변환·파인튜닝을 다루는 공식 서브 패키지.

```bash
pip install mlx-lm
```

#### mlx-lm — 설치 및 기본 사용

```bash
# 1. CLI 추론 — HuggingFace Hub 모델을 바로 실행
python -m mlx_lm.generate \
  --model mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --prompt "Explain attention mechanism in Korean" \
  --max-tokens 512 \
  --temp 0.7

# 2. 대화형 채팅 모드
python -m mlx_lm.generate \
  --model mlx-community/Qwen2.5-32B-Instruct-4bit \
  --chat
```

#### mlx-lm — OpenAI 호환 로컬 서버

```bash
# OpenAI API 규격 서버 시작 (기본 포트 8080)
python -m mlx_lm.server \
  --model mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --port 8080

# curl로 테스트
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mlx-community/Meta-Llama-3.1-70B-Instruct-4bit",
    "messages": [{"role": "user", "content": "안녕하세요"}],
    "max_tokens": 200
  }'
```

```python
# OpenAI SDK로 로컬 서버에 연결
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8080/v1", api_key="ignored")
response = client.chat.completions.create(
    model="mlx-community/Meta-Llama-3.1-70B-Instruct-4bit",
    messages=[{"role": "user", "content": "안녕하세요"}]
)
print(response.choices[0].message.content)
```

#### mlx-lm — Python API

```python
from mlx_lm import load, generate
from mlx_lm.utils import stream_generate

# 모델 로드 (첫 실행 시 HuggingFace에서 다운로드 → ~/.cache/huggingface)
model, tokenizer = load("mlx-community/Meta-Llama-3.1-70B-Instruct-4bit")

# 단순 생성
response = generate(
    model, tokenizer,
    prompt="한국어로 설명해줘: transformer architecture",
    max_tokens=512,
    temp=0.7,
    top_p=0.9,
)
print(response)

# 스트리밍 생성
for token in stream_generate(model, tokenizer, prompt="...", max_tokens=512):
    print(token, end="", flush=True)

# 채팅 형식 (apply_chat_template 사용)
messages = [
    {"role": "system", "content": "당신은 친절한 AI 어시스턴트입니다."},
    {"role": "user", "content": "Spring Boot의 장점은?"},
]
prompt = tokenizer.apply_chat_template(
    messages, tokenize=False, add_generation_prompt=True
)
response = generate(model, tokenizer, prompt=prompt, max_tokens=1024)
```

#### mlx-lm — HuggingFace 모델 변환 (convert)

HuggingFace의 일반 모델을 MLX 포맷(.npz)으로 변환하고 양자화 적용.

```bash
# FP16 → 4bit 양자화 변환 (Q4 ≈ 모델 크기 1/4)
python -m mlx_lm.convert \
  --hf-path meta-llama/Llama-3.1-70B-Instruct \
  --mlx-path ./llama-3.1-70b-mlx-4bit \
  --quantize \
  --q-bits 4

# 8bit 변환 (Q8 ≈ 고품질, 메모리 더 필요)
python -m mlx_lm.convert \
  --hf-path meta-llama/Llama-3.1-70B-Instruct \
  --mlx-path ./llama-3.1-70b-mlx-8bit \
  --quantize \
  --q-bits 8

# 변환 후 HuggingFace Hub 업로드
python -m mlx_lm.convert \
  --hf-path meta-llama/Llama-3.1-70B-Instruct \
  --mlx-path mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --quantize --q-bits 4 \
  --upload-repo mlx-community/Meta-Llama-3.1-70B-Instruct-4bit
```

| q-bits | 70B 모델 크기 | 메모리 요구량 | 품질 |
|---|---|---|---|
| 16 (FP16) | ~140GB | ~140GB | 최고 |
| 8 (Q8) | ~70GB | ~75GB | 높음 |
| 4 (Q4_K_M) | ~35GB | ~40GB | 실용적 |
| 3 (Q3) | ~26GB | ~30GB | 낮음 |

```bash
# Ollama로 즉시 LLM 추론 (Metal 백엔드 자동 활용)
brew install ollama
ollama pull llama3.1:70b
ollama run llama3.1:70b
```

### PyTorch MPS 백엔드

```python
import torch

# M5 Max GPU 사용
device = torch.device("mps" if torch.backends.mps.is_available() else "cpu")

model = MyModel().to(device)
x = torch.randn(64, 512).to(device)
output = model(x)   # Metal GPU에서 실행
```

> [!tip] MPS의 한계
> - `torch.compile()` MPS 백엔드 지원 제한적 (CUDA만큼 성숙하지 않음)
> - Triton 커널 미지원
> - 일부 연산자(Flash Attention 등) MPS fallback → CPU로 실행되어 성능 저하 가능

### CoreML 변환 및 추론

```python
import coremltools as ct
import torch

# PyTorch → CoreML 변환
model = MyPyTorchModel()
traced = torch.jit.trace(model, example_input)
mlmodel = ct.convert(
    traced,
    compute_units=ct.ComputeUnit.ALL  # CPU + GPU + Neural Engine 혼합
)
mlmodel.save("model.mlpackage")
```

---

## 5. LLM 추론 성능 (실측 참고)

| 모델 | 메모리 요구량 | 토큰/초 (Ollama Metal) | 비고 |
|---|---|---|---|
| Llama 3.1 8B (Q4_K_M) | ~5GB | ~60-80 tok/s | 실용적 속도 |
| Llama 3.1 70B (Q4_K_M) | ~40GB | ~8-12 tok/s | 128GB 여유 |
| Llama 3.1 70B (Q8) | ~75GB | ~5-7 tok/s | 고품질 양자화 |
| Llama 3.1 70B (FP16) | ~140GB | ❌ (128GB 초과) | 불가 |
| Llama 3.1 405B (Q4) | ~200GB | ❌ | 단독 불가 |
| Mistral 7B (Q4) | ~4GB | ~80-100 tok/s | 빠른 응답 |
| Qwen2.5 32B (Q4) | ~20GB | ~20-25 tok/s | 균형잡힌 성능 |

> 토큰/초 수치는 MacBook Pro 16" M4 Max 실측 기준. M5 Max는 약 10~15% 향상 예상.

---

## 6. 파인튜닝 — mlx-lm LoRA

#### 데이터셋 형식

```jsonl
// train.jsonl — chat 형식 (mlx-lm 기본)
{"messages": [{"role": "user", "content": "Q: ..."}, {"role": "assistant", "content": "A: ..."}]}
{"messages": [{"role": "user", "content": "Q: ..."}, {"role": "assistant", "content": "A: ..."}]}
```

```
data/
  train.jsonl
  valid.jsonl     # 검증셋 (선택, 없으면 train에서 분할)
```

#### LoRA 학습

```bash
# LoRA 파인튜닝 실행
python -m mlx_lm.lora \
  --model mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --train \
  --data ./data \
  --iters 2000 \
  --lora-layers 16 \       # 상위 N개 레이어에 LoRA 적용
  --batch-size 4 \
  --learning-rate 1e-4 \
  --steps-per-eval 100 \
  --save-every 500 \
  --adapter-path ./adapters  # 어댑터 저장 위치
```

| 파라미터 | 권장값 | 설명 |
|---|---|---|
| `--lora-layers` | 8~32 | 많을수록 품질↑ 메모리↑ |
| `--batch-size` | 2~8 | 128GB → 4~8 가능 |
| `--iters` | 1000~5000 | 데이터 크기에 따라 조절 |
| `--learning-rate` | 1e-5 ~ 1e-4 | 작게 시작 권장 |

#### LoRA 추론 (어댑터 적용)

```bash
# 어댑터를 붙여서 추론
python -m mlx_lm.generate \
  --model mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --adapter-path ./adapters \
  --prompt "파인튜닝된 모델 테스트"
```

```python
from mlx_lm import load, generate

# 어댑터 포함 로드
model, tokenizer = load(
    "mlx-community/Meta-Llama-3.1-70B-Instruct-4bit",
    adapter_path="./adapters"
)
response = generate(model, tokenizer, prompt="...", max_tokens=512)
```

#### 어댑터 병합 (fuse) — 배포용 단일 모델 생성

```bash
# LoRA 어댑터를 베이스 모델에 병합 → 단일 MLX 모델
python -m mlx_lm.fuse \
  --model mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --adapter-path ./adapters \
  --save-path ./fused-model \
  --de-quantize          # 선택: 병합 시 역양자화 (FP16으로 저장)

# 병합 후 재양자화 (배포 크기 최소화)
python -m mlx_lm.convert \
  --hf-path ./fused-model \
  --mlx-path ./fused-model-4bit \
  --quantize --q-bits 4
```

> **128GB 통합 메모리의 장점**: 70B 모델 LoRA 파인튜닝 시 모델 가중치 + Optimizer state + 배치 데이터가 동일 메모리에 상주. CPU↔GPU 스왑 없이 안정적.

---

## 7. 개발 환경 설정

```bash
# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Python ML 환경 (ARM 네이티브)
brew install miniforge
conda create -n ml python=3.11
conda activate ml

# MLX (Apple Silicon 네이티브)
pip install mlx mlx-lm

# PyTorch (MPS 지원)
pip install torch torchvision torchaudio

# HuggingFace
pip install transformers datasets accelerate

# CoreML
pip install coremltools

# LLM 서버
brew install ollama
```

---

## 8. 장점 / 단점

### ✅ 장점
- **~600+ GB/s 메모리 대역폭** — LLM 추론 배치 처리 속도에서 DGX Spark(273 GB/s) 대비 2배 이상
- **zero-copy UMA** — CPU↔GPU 메모리 복사 오버헤드 없음
- **전력 효율** — ~92W TDP, LPDDR5X 전력 효율성 → 배터리 구동 가능
- **macOS 생태계** — Xcode, Safari, Office 등 생산성 도구와 AI 개발 환경 병행
- **MLX 프레임워크** — Apple이 직접 관리하는 Apple Silicon 특화 ML 프레임워크
- **Ollama / llama.cpp Metal** — 70B 모델을 낮은 세팅 부담으로 즉시 실행
- **조용한 냉각** — 데스크탑 급 성능을 저소음으로

### ❌ 단점
- **CUDA 미지원** — PyTorch GPU 가속이 MPS로 제한, cuDNN·TensorRT·vLLM 등 CUDA 라이브러리 직접 불가
- **AI FLOPS 절대값** — ~45 TFLOPS(FP16) vs DGX Spark 250 TFLOPS(FP16) → 대규모 학습 불가
- **MPS 미성숙** — PyTorch MPS 백엔드가 CUDA만큼 연산자 커버리지·최적화가 안 됨
- **스토리지 속도** — ~8 GB/s (DGX Spark PCIe Gen5 ~14 GB/s 대비 느림)
- **메모리 확장 불가** — SoC 통합으로 구매 시점에 용량 고정
- **Linux 지원 미흡** — Asahi Linux GPU 드라이버 미완성, 서버 배포 사실상 불가

---

## 9. AI 워크로드별 적합성

| 워크로드 | M5 Max | DGX Spark | 비고 |
|---|---|---|---|
| LLM 추론 (단일 사용자) | ✅ 우수 | ✅ 가능 | M5 Max 대역폭 유리 |
| LLM 추론 (고배치) | ❌ | ✅ 우수 | FLOPS 차이 |
| LLM LoRA 파인튜닝 (70B) | ✅ 가능 (MLX) | ✅ 가능 (CUDA) | 속도는 DGX 유리 |
| 풀 파인튜닝 (70B+) | ❌ | ⚠️ 제한적 | 클라우드 필요 |
| 비전/분류 (CoreML) | ✅ Neural Engine | ❌ | CoreML 온디바이스 |
| 음성 인식 (Whisper) | ✅ 매우 빠름 | ✅ | 둘 다 우수 |
| 이미지 생성 (SD) | ✅ Metal ANE | ✅ CUDA | 방식 상이 |
| 개발·실험 환경 | ✅ macOS 편의성 | ⚠️ Linux only | 생산성 차이 |
| 프로덕션 추론 서버 | ❌ macOS 제한 | ✅ Docker/K8s | 서버화 불가 |

---

## 10. 관련
- [[DGX-Spark]] · [[Single-Local-AI-Machine]] (단일 장비 선택 비교) · [[_index]]
