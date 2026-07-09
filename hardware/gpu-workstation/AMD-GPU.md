---
tags:
  - hardware
  - gpu
  - amd
  - rocm
  - ai
created: 2026-06-16
---

# AMD GPU — AI/HPC 라인업 & ROCm 스택

> [!summary] 한 줄 요약
> AMD는 CDNA 아키텍처 데이터센터 GPU(MI300X)와 RDNA 소비자 GPU로 NVIDIA에 도전 중. **MI300X**는 192GB HBM3로 H100 80GB를 메모리에서 2.4배 앞서며, ROCm 소프트웨어 스택이 성숙해지며 LLM 추론 서버 대안으로 부상.

---

## 1. AMD GPU 라인업 비교

### 데이터센터 — CDNA 아키텍처

| 모델 | 아키텍처 | 메모리 | 메모리 BW | FP16 TFLOPS | 출시 |
|------|---------|--------|----------|------------|------|
| **MI300X** | CDNA 3 | 192GB HBM3 | 5.3 TB/s | 1,307 | 2024 |
| **MI300A** | CDNA 3 | 128GB HBM3 (CPU+GPU 통합) | 5.3 TB/s | 765 | 2024 |
| **MI250X** | CDNA 2 | 128GB HBM2e | 3.2 TB/s | 383 | 2022 |
| **MI100** | CDNA 1 | 32GB HBM2e | 1.2 TB/s | 184 | 2020 |

### 워크스테이션/소비자 — RDNA 아키텍처

| 모델 | 아키텍처 | 메모리 | 용도 |
|------|---------|--------|------|
| Radeon RX 7900 XTX | RDNA 3 | 24GB GDDR6 | 게임 / 간단한 AI |
| Radeon PRO W7900 | RDNA 3 | 48GB GDDR6 | 워크스테이션 |
| Radeon PRO W7800 | RDNA 3 | 32GB GDDR6 | 워크스테이션 |

---

## 2. MI300X vs H100 상세 비교

| 항목 | AMD MI300X | NVIDIA H100 SXM5 |
|------|-----------|----------------|
| 아키텍처 | CDNA 3 | Hopper |
| 메모리 | **192GB HBM3** | 80GB HBM3 |
| 메모리 BW | 5.3 TB/s | 3.35 TB/s |
| FP16 TFLOPS | 1,307 | 989 |
| FP8 TFLOPS | 2,614 | 1,979 |
| NVLink/Infinity Fabric | Infinity Fabric 896 GB/s | NVLink 4.0 900 GB/s |
| MIG 지원 | CPX (Compute Partitioning) | 최대 7 인스턴스 |
| TDP | 750W | 700W |
| 가격대 | ~$10,000~15,000 | ~$20,000~30,000 |

**MI300X 핵심 강점**: 192GB VRAM → **405B 파라미터 모델 FP4로 단일 GPU 배치 가능** (H100 필요 GPU 수의 절반 이하).

---

## 3. ROCm 소프트웨어 스택

```
[CUDA 생태계]          [ROCm 생태계]
CUDA Toolkit     ←→    ROCm (HIP)
cuDNN            ←→    MIOpen
NCCL             ←→    RCCL
cuBLAS           ←→    rocBLAS
Thrust           ←→    rocThrust
TensorRT         ←→    MIGraphX
```

### 설치 (Ubuntu 22.04)

```bash
# AMD 공식 설치 스크립트
wget https://repo.radeon.com/amdgpu-install/6.1/ubuntu/jammy/amdgpu-install_6.1.60100-1_all.deb
dpkg -i amdgpu-install_6.1.60100-1_all.deb
amdgpu-install --usecase=rocm

# 사용자를 render/video 그룹에 추가
usermod -aG render,video $LOGNAME

# 확인
rocm-smi             # GPU 상태 모니터링 (nvidia-smi 대응)
rocminfo             # GPU 상세 정보
hipcc --version      # HIP 컴파일러 버전
```

### PyTorch on ROCm

```bash
# ROCm 전용 PyTorch 설치 (pip)
pip install torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/rocm6.0

# 확인
python -c "import torch; print(torch.cuda.is_available())"  # True
python -c "import torch; print(torch.version.hip)"          # ROCm 버전
```

---

## 4. vLLM on ROCm (LLM 서버)

```bash
# Docker로 vLLM ROCm 실행 (가장 안정적)
docker pull rocm/vllm:latest

docker run --device /dev/kfd --device /dev/dri \
  --group-add render \
  --ipc=host \
  -p 8000:8000 \
  -v $HF_HOME:/root/.cache/huggingface \
  rocm/vllm:latest \
  python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-70B-Instruct \
    --tensor-parallel-size 2      # MI300X 2개 병렬
```

```python
# HIP (ROCm용 CUDA 이식 레이어) 사용 확인
import torch
print(torch.cuda.get_device_name(0))     # AMD Instinct MI300X
print(torch.cuda.get_device_properties(0).total_memory // (1024**3), "GB")  # 192
```

---

## 5. 클라우드에서 MI300X 접근

| 클라우드 | 인스턴스 | 비고 |
|---------|---------|------|
| **Microsoft Azure** | ND MI300X v5 | 8× MI300X, Azure OpenAI 내부 사용 |
| **Oracle Cloud** | BM.GPU.MI300X.8 | 8× MI300X, 베어메탈 |
| **Lambda Labs** | 8× MI300X | 시간당 과금 |
| AWS | 미지원 (자체 Trainium/Inferentia) | — |

---

## 6. NVIDIA vs AMD 선택 기준

| 기준 | NVIDIA 선택 | AMD 선택 |
|------|-----------|---------|
| **소프트웨어 호환성** | CUDA 생태계 (압도적) | ROCm 성숙도 향상 중 |
| **대형 모델 배치** | H200(141GB) 필요 | MI300X(192GB) 단일 GPU로 커버 |
| **비용** | 높음 | 상대적 저렴 (MI300X vs H100) |
| **미세 조정** | 안정적 | PyTorch ROCm은 대부분 지원 |
| **TensorRT 최적화** | 지원 | MIGraphX (성숙도 낮음) |
| **기업 지원** | 강함 | Azure/Oracle에서 확대 중 |

---

## 7. rocm-smi — GPU 모니터링

```bash
# nvidia-smi 대응 명령어
rocm-smi                          # 전체 GPU 요약
rocm-smi --showmeminfo vram       # VRAM 사용량
rocm-smi --showtemp               # 온도
rocm-smi --showpower              # 전력 소비
rocm-smi --showclocks             # 클럭 속도
rocm-smi --showpids               # GPU 사용 중인 프로세스
watch -n 1 rocm-smi               # 실시간 모니터링
```

---

## 8. 관련
- [[GPU-Specs]] — NVIDIA GPU 스펙 비교
- [[../../ai/Inference-Optimization]] — vLLM, 모델 양자화
- [[../../ai/Fine-Tuning]] — ROCm에서 LoRA 학습
