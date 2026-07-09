---
tags:
  - hardware
  - gpu
  - nvidia
  - specs
  - ai
created: 2026-06-16
---

# NVIDIA GPU 스펙 비교

> [!summary] 한 줄 요약
> NVIDIA GPU 라인업 스펙 정리. **소비자용(GeForce)**, **워크스테이션 레거시(Ampere RTX A)**, **데이터센터 추론(L40S · A100 · H100 · H200 · B200)**, **워크스테이션 신규(RTX Pro Blackwell)** 카테고리. LLM 실행 시 **VRAM 크기**와 **메모리 대역폭**이 핵심 지표.

---

## 1. 카테고리 정리

```
[소비자용 GeForce]         게이밍 + 개인 AI 작업, NVLink 없음
  RTX 4080 / 4090 (Ada Lovelace)
  RTX 5080 / 5090 (Blackwell)

[워크스테이션 레거시]        전문가용, ECC 메모리, ISV 인증
  RTX A4000 / A6000 (Ampere)

[데이터센터 추론/HPC]        서버 랙 탑재, NVLink, HBM, MIG
  L40S (Ada Lovelace)        — 추론 특화, PCIe
  A100 (Ampere)              — HBM2e, NVLink 3.0, MIG
  H100 (Hopper)              — HBM3, FP8 Transformer Engine, NVLink 4.0
  H200 (Hopper + HBM3e)     — H100 컴퓨트 + 141GB HBM3e 업그레이드
  B200 (Blackwell)           — 192GB HBM3e, FP4, NVLink 5.0

[워크스테이션 신규]          전문가용 Blackwell, FP4 지원
  RTX Pro 2000 / 4000 / 6000 Blackwell
```

---

## 2. 소비자용 GeForce (Ada Lovelace)

### RTX 4080

| 항목 | 값 |
|---|---|
| **아키텍처** | Ada Lovelace (AD103) |
| **CUDA 코어** | 9,728 |
| **Tensor 코어** | 304 (4세대) |
| **VRAM** | 16 GB GDDR6X |
| **메모리 버스** | 256-bit |
| **메모리 대역폭** | 716 GB/s |
| **FP32 성능** | 48.7 TFLOPS |
| **FP16 / BF16** | 97.4 TFLOPS |
| **TDP** | 320 W |
| **출시** | 2022년 11월 |

**LLM 적합도**: 16GB VRAM → 7B Q4 모델 2개, 또는 13B Q4 1개 수용.

---

### RTX 4090

| 항목 | 값 |
|---|---|
| **아키텍처** | Ada Lovelace (AD102) |
| **CUDA 코어** | 16,384 |
| **Tensor 코어** | 512 (4세대) |
| **VRAM** | 24 GB GDDR6X |
| **메모리 버스** | 384-bit |
| **메모리 대역폭** | 1,008 GB/s |
| **FP32 성능** | 82.6 TFLOPS |
| **FP16 / BF16** | 165 TFLOPS |
| **TDP** | 450 W |
| **출시** | 2022년 10월 |

**LLM 적합도**: 24GB VRAM → 13B Q4 로 여유, 34B Q4 가능. 개인 로컬 LLM 사실상 최고 가성비(Ada 세대).

---

## 3. 소비자용 GeForce (Blackwell)

### RTX 5080

| 항목 | 값 |
|---|---|
| **아키텍처** | Blackwell (GB203) |
| **CUDA 코어** | 10,752 |
| **Tensor 코어** | 336 (5세대) |
| **RT 코어** | 84 (4세대) |
| **VRAM** | 16 GB GDDR7 |
| **메모리 버스** | 256-bit |
| **메모리 대역폭** | 960 GB/s |
| **FP16 / BF16** | ~112 TFLOPS |
| **AI TOPS (INT8)** | 1,801 |
| **FP4 지원** | ✅ (Blackwell 전용) |
| **TDP** | 400 W |
| **출시** | 2025년 1월 |

**특이점**: VRAM이 4090과 동일한 16GB이지만 대역폭은 960 GB/s로 4090(1,008 GB/s)과 유사. FP4 양자화 추론 지원으로 단위 대역폭당 처리량↑.

---

### RTX 5090

| 항목 | 값 |
|---|---|
| **아키텍처** | Blackwell (GB202) |
| **CUDA 코어** | 21,760 |
| **Tensor 코어** | 680 (5세대) |
| **RT 코어** | 170 (4세대) |
| **VRAM** | 32 GB GDDR7 |
| **메모리 버스** | 512-bit |
| **메모리 대역폭** | 1,792 GB/s |
| **FP32 성능** | ~104.8 TFLOPS |
| **FP16 Tensor** | ~209 TFLOPS (dense) / ~419 TFLOPS (sparse) |
| **AI TOPS (FP4 sparse)** | 3,352 |
| **FP4 지원** | ✅ |
| **TDP** | 575 W |
| **출시** | 2025년 1월 |

**LLM 적합도**: 32GB VRAM → 70B Q4 단일 GPU 실행 가능. 메모리 대역폭 1,792 GB/s은 A100 SXM(2,000 GB/s) 수준. 개인용 최고 사양.

---

## 4. 워크스테이션 레거시 (Ampere)

### RTX A4000

| 항목 | 값 |
|---|---|
| **아키텍처** | Ampere (GA104) |
| **CUDA 코어** | 6,144 |
| **VRAM** | 16 GB GDDR6 ECC |
| **메모리 버스** | 256-bit |
| **메모리 대역폭** | 448 GB/s |
| **FP32 성능** | 19.2 TFLOPS |
| **TDP** | 140 W |
| **ECC** | ✅ |
| **NVLink** | ✅ (2-way) |
| **출시** | 2021년 4월 |

**용도**: 저전력(140W) 워크스테이션 탑재, ISV 인증. AI 추론보다 CAD/시각화 용도. 대역폭이 낮아 LLM은 소형만 실용.

---

### RTX A6000

| 항목 | 값 |
|---|---|
| **아키텍처** | Ampere (GA102) |
| **CUDA 코어** | 10,752 |
| **VRAM** | 48 GB GDDR6 ECC |
| **메모리 버스** | 384-bit |
| **메모리 대역폭** | 768 GB/s |
| **FP32 성능** | 38.7 TFLOPS |
| **TDP** | 300 W |
| **ECC** | ✅ |
| **NVLink** | ✅ (2-way, 2개 연결 시 96GB) |
| **출시** | 2020년 10월 |

**LLM 적합도**: 48GB VRAM → 33B FP16 전체, 70B Q4 가능. NVLink 2-way로 96GB 구성 가능 (70B BF16 전체). 안정적인 워크스테이션 AI 플랫폼.

---

## 5. 데이터센터 추론 (Ada Lovelace)

### L40S

| 항목 | 값 |
|---|---|
| **아키텍처** | Ada Lovelace (AD102) |
| **CUDA 코어** | 18,176 |
| **Tensor 코어** | 568 (4세대) |
| **VRAM** | 48 GB GDDR6 ECC |
| **메모리 버스** | 384-bit |
| **메모리 대역폭** | 864 GB/s |
| **FP32 성능** | 91.6 TFLOPS |
| **FP16 Tensor** | 362 TFLOPS |
| **FP8 Tensor** | 724 TFLOPS |
| **TDP** | 350 W |
| **폼팩터** | PCIe (패시브 쿨링, 서버용) |
| **ECC** | ✅ |
| **출시** | 2023년 9월 |

**용도**: 클라우드 AI 추론 서버. 48GB ECC + FP8 지원으로 서버 추론 워크로드 최적화. vLLM + PagedAttention 운영에 이상적.

```
L40S vs RTX 4090 비교 (추론 관점):
  VRAM:  L40S 48GB  vs 4090 24GB  → L40S 압도적
  대역폭: L40S 864  vs 4090 1,008 → 4090 약간 우위
  FP16:  L40S 362  vs 4090 165   → L40S 2.2배
  냉각:  L40S 패시브(서버랙) vs 4090 액티브(소비자)
  가격:  L40S ~$15K vs 4090 ~$2K → 서버 규모 투자
```

---

## 6. 데이터센터 HPC (Ampere → Hopper → Blackwell)

---

### A100

| 항목 | PCIe (80GB) | SXM4 (80GB) |
|---|---|---|
| **아키텍처** | Ampere (GA100) | Ampere (GA100) |
| **CUDA 코어** | 6,912 | 6,912 |
| **Tensor 코어** | 432 (3세대) | 432 (3세대) |
| **VRAM** | 80 GB HBM2e | 80 GB HBM2e |
| **메모리 대역폭** | 1,935 GB/s | 2,039 GB/s |
| **FP16 Tensor** | 312 TFLOPS (624 sparse) | 312 TFLOPS (624 sparse) |
| **FP32** | 19.5 TFLOPS | 19.5 TFLOPS |
| **FP8** | ❌ | ❌ |
| **TF32** | 156 TFLOPS (312 sparse) | 156 TFLOPS (312 sparse) |
| **NVLink** | ❌ | NVLink 3.0, 600 GB/s |
| **TDP** | 300 W | 400 W |
| **MIG** | ✅ (최대 7 인스턴스) | ✅ |
| **ECC** | ✅ | ✅ |
| **출시** | 2020년 | 2020년 |

**A100 특징**:
- **MIG (Multi-Instance GPU)**: 단일 GPU를 최대 7개의 독립 인스턴스로 분할 → 여러 소규모 작업 병렬 실행
- 40GB 버전도 존재 (HBM2, 1,555 GB/s, TDP 300W)
- A800: 중국 수출용, NVLink 400 GB/s 제한 버전

```
LLM 적합도 (80GB):
  70B BF16 전체 단일 GPU 불가 (80GB ≒ 140GB 필요)
  70B Q4 ≈ 35~40GB → 단일 GPU 가능
  A100×2 NVLink → 160GB → 70B BF16 전체 운용
```

---

### H100

| 항목 | PCIe (80GB) | SXM5 (80GB) |
|---|---|---|
| **아키텍처** | Hopper (GH100) | Hopper (GH100) |
| **CUDA 코어** | 14,592 | 16,896 |
| **Tensor 코어** | 456 (4세대) | 528 (4세대) |
| **VRAM** | 80 GB HBM2e | 80 GB HBM3 |
| **메모리 대역폭** | 2,000 GB/s | 3,350 GB/s |
| **FP16 Tensor** | 756 TFLOPS (1,513 sparse) | 989 TFLOPS (1,979 sparse) |
| **FP8 Tensor** | 1,513 TFLOPS (3,026 sparse) | 1,979 TFLOPS (3,958 sparse) |
| **FP32** | ~51 TFLOPS | ~67 TFLOPS |
| **FP8** | ✅ (Transformer Engine) | ✅ |
| **NVLink** | ❌ | NVLink 4.0, 900 GB/s |
| **TDP** | 350 W | 700 W |
| **MIG** | ✅ | ✅ |
| **ECC** | ✅ | ✅ |
| **출시** | 2022년 | 2022년 |

**H100 특징**:
- **Transformer Engine**: FP8↔FP16을 레이어 단위로 동적 전환 → LLM 학습·추론 최적화
- SXM5가 PCIe 대비 메모리 대역폭 67% 우세 (3,350 vs 2,000 GB/s)
- H800: 중국 수출용, NVLink 400 GB/s 제한

---

### H200

| 항목 | SXM5 (141GB) |
|---|---|
| **아키텍처** | Hopper (GH100) — H100 동일 컴퓨트 |
| **CUDA 코어** | 16,896 |
| **Tensor 코어** | 528 (4세대) |
| **VRAM** | **141 GB HBM3e** |
| **메모리 대역폭** | **4,800 GB/s** |
| **FP16 Tensor** | 989 TFLOPS (H100 동일) |
| **FP8 Tensor** | 1,979 TFLOPS (H100 동일) |
| **NVLink** | NVLink 4.0, 900 GB/s |
| **TDP** | 700 W (H100 동일) |
| **출시** | 2023년 말 |

**H200 특징**:
- H100 대비 **컴퓨트는 동일**, VRAM 80GB → **141GB**, 대역폭 3,350 → **4,800 GB/s**
- 추가 전력 소모 없이 메모리 제약 해소 → 메모리 바운드 추론 워크로드에 최적
- 70B BF16 단일 GPU 이론 가능 (141GB ÷ ~140GB 필요 ≒ 아슬하게 가능)
- GH200: Grace CPU (72코어 Arm Neoverse V2) + H100/H200 통합 패키지

```
H100 vs H200 선택 기준:
  컴퓨트 병목이면   → H100 (가격 낮음, 컴퓨트 동일)
  메모리 병목이면   → H200 (더 큰 모델 수용, 대역폭 +43%)
```

---

### B200

| 항목 | SXM6 |
|---|---|
| **아키텍처** | Blackwell (GB100, 이중 다이) |
| **CUDA 코어** | ~20,480 (추정, dual-die) |
| **Tensor 코어** | 5세대 (FP4 네이티브) |
| **VRAM** | **192 GB HBM3e** |
| **메모리 대역폭** | **8,000 GB/s** |
| **FP16 Tensor** | ~4,500 TFLOPS |
| **FP8 Tensor** | **9,000 TFLOPS** (dense) |
| **FP4 Tensor** | **18,000 TFLOPS** (sparse) |
| **FP32** | ~80 TFLOPS |
| **FP4** | ✅ (5세대 Tensor Core, 네이티브) |
| **NVLink** | NVLink 5.0, **1,800 GB/s** |
| **TDP** | **1,000 W** |
| **출시** | 2025년 (GB200 NVL72 시스템으로 배포) |

**B200 특징**:
- H100 대비 FP8 성능 **4.5배**, 메모리 대역폭 **2.4배**
- 192GB → 405B 모델 FP4 단일 GPU 이론 가능
- **GB200 NVL72**: 36 Grace CPU + 72 B200 GPU 랙 시스템, NVLink 5.0 풀메시 연결

```
세대별 FP8 성능 비교 (SXM 기준):
  A100 : FP8 미지원
  H100 : 1,979 TFLOPS FP8 dense
  H200 : 1,979 TFLOPS FP8 dense (H100 동일)
  B200 : 9,000 TFLOPS FP8 dense  ← 4.5배 도약
```

---

## 7. 워크스테이션 신규 (RTX Pro Blackwell)

### RTX Pro 2000 Blackwell

| 항목 | 값 |
|---|---|
| **아키텍처** | Blackwell (GB205) |
| **CUDA 코어** | 4,352 |
| **Tensor 코어** | 136 (5세대) |
| **VRAM** | 16 GB GDDR7 ECC |
| **메모리 대역폭** | 288 GB/s |
| **FP4 지원** | ✅ |
| **TDP** | ~75 W |
| **폼팩터** | Single-slot, 저전력 |
| **ECC** | ✅ |
| **출시** | 2025년 3월 |

**용도**: 소형 워크스테이션·슬림 PC 탑재. 전력 제약이 있는 환경에서 Blackwell FP4 추론 가능.

---

### RTX Pro 4000 Blackwell

| 항목 | 값 |
|---|---|
| **아키텍처** | Blackwell |
| **CUDA 코어** | 8,960 |
| **Tensor 코어** | 280 (5세대) |
| **RT 코어** | 70 (4세대) |
| **VRAM** | 24 GB GDDR7 ECC |
| **FP4 지원** | ✅ |
| **ECC** | ✅ |
| **가격 (참고)** | ~$1,546 |
| **출시** | 2025년 3월 |

**용도**: 중급 워크스테이션. 24GB ECC + FP4 → 70B Q4 단일 GPU 가능 (Pro 6000 대비 저비용).

---

### RTX Pro 6000 Blackwell

| 항목 | 값 |
|---|---|
| **아키텍처** | Blackwell (GB102) |
| **CUDA 코어** | 24,064 |
| **Tensor 코어** | 752 (5세대) |
| **RT 코어** | 188 (4세대) |
| **VRAM** | 96 GB GDDR7 ECC |
| **메모리 버스** | 384-bit |
| **메모리 대역폭** | 1,792 GB/s |
| **AI TOPS** | 4,000 |
| **FP4 지원** | ✅ |
| **ECC** | ✅ |
| **NVLink** | ✅ (2-way, 192GB 구성 가능) |
| **가격 (참고)** | ~$8,565 |
| **출시** | 2025년 3월 |

**LLM 적합도**: 96GB VRAM → 70B BF16 전체 단일 GPU 수용. NVLink 2-way로 192GB → 405B 모델도 이론 가능. 워크스테이션 LLM 단일 GPU 최강.

---

## 8. 종합 비교표

### VRAM 기준 (LLM 모델 수용 가이드)

| GPU | VRAM | 메모리 타입 | 수용 가능 모델 (대략) |
|---|---|---|---|
| RTX 4080 / 5080 | 16 GB | GDDR6X / GDDR7 | 7B Q4, 13B Q4 |
| RTX 4090 / RTX Pro 4000 | 24 GB | GDDR6X / GDDR7 | 34B Q4, 13B BF16 |
| RTX 5090 | 32 GB | GDDR7 | 34B BF16, 70B Q4 |
| A100 | 40 / 80 GB | HBM2 / HBM2e | 70B Q4 (80GB) |
| RTX A6000 / L40S | 48 GB | GDDR6 / GDDR6 ECC | 33B BF16, 70B Q4 여유 |
| H100 / H100 SXM5 | 80 GB | HBM2e / HBM3 | 70B BF16 (빡빡) |
| H200 | 141 GB | HBM3e | **70B BF16 여유**, 405B Q4 |
| RTX Pro 6000 Blackwell | 96 GB | GDDR7 ECC | 70B BF16 전체 ⭐ |
| B200 | 192 GB | HBM3e | **405B Q4**, 70B BF16 × 2 |
| RTX Pro 6000 × 2 (NVLink) | 192 GB | GDDR7 ECC | 405B Q4 이론 가능 |

### 메모리 대역폭 기준 (토큰 생성 속도 — 높을수록 빠름)

| GPU | 메모리 BW | 세대 | 카테고리 |
|---|---|---|---|
| B200 | **8,000 GB/s** | Blackwell | HPC |
| H200 | **4,800 GB/s** | Hopper | HPC |
| H100 SXM5 | 3,350 GB/s | Hopper | HPC |
| H100 PCIe | 2,000 GB/s | Hopper | HPC |
| A100 SXM4 | 2,039 GB/s | Ampere | HPC |
| RTX Pro 6000 / RTX 5090 | 1,792 GB/s | Blackwell | 워크스테이션 / 소비자 |
| RTX 4090 | 1,008 GB/s | Ada | 소비자 |
| RTX 5080 | 960 GB/s | Blackwell | 소비자 |
| L40S | 864 GB/s | Ada | 데이터센터 추론 |
| RTX A6000 | 768 GB/s | Ampere | 워크스테이션 |
| A100 PCIe | 1,935 GB/s | Ampere | HPC |
| RTX A4000 | 448 GB/s | Ampere | 워크스테이션 |
| RTX Pro 2000 Blackwell | 288 GB/s | Blackwell | 워크스테이션 저전력 |

### 용도별 선택 가이드

| 용도 | 권장 GPU | 이유 |
|---|---|---|
| **개인 로컬 LLM (최고 성능)** | RTX 5090 | 32GB GDDR7, 1,792 GB/s, FP4 |
| **개인 로컬 LLM (가성비)** | RTX 4090 | 24GB, 1,008 GB/s, 중고 가격 유리 |
| **70B BF16 단일 GPU (워크스테이션)** | RTX Pro 6000 Blackwell | 96GB ECC |
| **70B BF16 단일 GPU (서버)** | H200 | 141GB HBM3e, 4,800 GB/s |
| **대규모 학습·추론 (서버)** | H100 SXM5 | 3,350 GB/s, NVLink 4.0, FP8 |
| **최대 성능 (서버)** | B200 | 192GB, 8,000 GB/s, FP4, NVLink 5.0 |
| **서버 추론 최적화** | L40S | 48GB ECC, FP8, PCIe 패시브 쿨링 |
| **워크스테이션 AI + 창작** | RTX Pro 4000 / Pro 6000 Blackwell | ECC, ISV 인증, FP4 |
| **저전력 워크스테이션** | RTX Pro 2000 Blackwell | ~75W, 슬림 폼팩터 |
| **레거시 서버 (Ampere)** | A100 / RTX A6000 | 검증된 CUDA 생태계, MIG |

---

## 9. AI 연산 정밀도별 지원 현황

| GPU | FP32 | FP16 / BF16 | TF32 | FP8 | FP4 | MIG |
|---|---|---|---|---|---|---|
| RTX 4080 / 4090 | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| RTX A4000 / A6000 | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **A100** | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| L40S | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **H100 / H200** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| RTX 5080 / 5090 | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| RTX Pro Blackwell 全 | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **B200** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

> **TF32**: Tensor Float 32 — NVIDIA A100부터 도입. FP32 범위 + FP16 수준 처리속도. LLM 학습 기본 정밀도.  
> **FP8**: H100부터 네이티브 지원 (Transformer Engine). 추론 속도 FP16 대비 2배.  
> **FP4**: Blackwell 전용. 동일 VRAM에서 처리 가능한 배치 크기 대폭 증가.

---

## 10. 관련
- [[DGX-Spark]] · [[Apple-M5-Max]] · [[Inference-Optimization]] · [[SLLM]]
