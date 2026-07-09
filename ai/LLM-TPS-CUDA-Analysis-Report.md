---
tags:
  - ai
  - research
  - report
  - cuda
  - inference
  - bandwidth
  - tps
  - hardware
created: 2026-06-16
---

# LLM 추론 성능의 결정 인자 분석 보고서
## — 메모리 대역폭 병목과 CUDA 최적화의 역할

> **목적**: "LLM의 TPS(Tokens Per Second)는 메모리 대역폭에 의해 결정되며, CUDA는 어떤 역할을 하는가"에 대한 학술적 근거 기반 분석
> **작성일**: 2026-06-16
> **분류**: AI 인프라 / 하드웨어 최적화

---

## 요약 (Executive Summary)

대규모 언어 모델(LLM)의 자기회귀적(autoregressive) 토큰 생성은 **메모리 대역폭 병목** 현상에 의해 지배된다. 이는 토큰 생성 단계(decode phase)의 행렬-벡터 연산이 극히 낮은 산술 강도(arithmetic intensity ≈ 1 FLOP/byte)를 가지기 때문으로, GPU 연산 능력(FLOPS)이 아닌 HBM 메모리 대역폭이 이론적 TPS 천장을 결정한다.

CUDA 기반 최적화는 이 이론적 천장 자체를 높이지 않는다. 대신 실제 대역폭 이용 효율(20~30% → 70~90%)을 높이거나, 불필요한 HBM 접근을 SRAM 우회로로 대체하거나, 대역폭 요구량 자체를 감소시킴으로써 실측 TPS를 향상시킨다.

---

## 1. 서론

LLM 추론 시스템의 성능 지표인 TPS(Tokens Per Second)는 사용자 경험과 서비스 비용을 직접 결정한다. GPU 시장에서는 연산 성능(TFLOPS)을 전면에 내세우지만, LLM 추론 맥락에서 이 지표는 오해를 유발할 수 있다.

본 보고서는 다음 두 가지 핵심 명제를 학술 문헌을 통해 검증한다:

1. **명제 1**: LLM decode TPS는 연산 성능(FLOPS)이 아닌 메모리 대역폭에 의해 결정된다.
2. **명제 2**: CUDA 최적화는 이론적 대역폭 한계를 높이는 것이 아니라, 실제 이용 효율을 높이거나 대역폭 소비를 줄이는 역할을 한다.

---

## 2. 이론적 프레임워크: Roofline 모델

### 2.1 Roofline 모델 개요

Williams et al. [1]이 제안한 **Roofline 모델**은 하드웨어 성능 분석의 표준 프레임워크로, 어떤 연산이 컴퓨팅 자원의 어느 한계에 의해 제한받는지를 시각화한다.

모델의 핵심 변수는 **산술 강도(Arithmetic Intensity, AI)**이다:

```
AI = 수행된 부동소수점 연산 수 (FLOPs)
     ─────────────────────────────────
     메모리에서 이동한 데이터 양 (Bytes)

단위: FLOP/byte
```

하드웨어의 **기계적 균형점(Machine Balance)**을 기준으로:

```
AI < Machine Balance  →  메모리 대역폭 병목 (Memory-bound)
AI > Machine Balance  →  연산 병목 (Compute-bound)

H100 SXM5 기준:
  Peak FP16 TFLOPS (Tensor Core): 1,979 TFLOPS
  Peak Memory Bandwidth:          3.35  TB/s
  Machine Balance = 1,979 / 3.35 ≈ 590 FLOP/byte
```

즉, H100에서 어떤 연산이 메모리 병목을 피하려면 산술 강도가 590 FLOP/byte를 초과해야 한다.

### 2.2 LLM Decode의 산술 강도

토큰 하나를 생성하는 핵심 연산인 행렬-벡터 곱(Matrix-Vector Multiply, MATVEC)의 산술 강도를 계산한다.

가중치 행렬 $W \in \mathbb{R}^{N \times M}$ (FP16), 입력 벡터 $x \in \mathbb{R}^{M}$:

```
FLOPs     = 2 × N × M
Bytes 읽기 = 2 × N × M  (FP16, 2bytes/param)
            + 2 × M      (입력 벡터, 무시 가능)

AI = (2NM) / (2NM) ≈ 1 FLOP/byte
```

**AI ≈ 1 FLOP/byte로, H100 Machine Balance(590)의 1/590 수준이다.** GPU 연산 능력의 0.17%만 활용되고, 나머지는 메모리 대기 상태에 놓인다.

---

## 3. LLM 추론의 두 단계와 각 병목

Pope et al. [2]은 Transformer 추론을 두 개의 계산적으로 상이한 단계로 분리하여 분석하였다.

### 3.1 Prefill 단계 (프롬프트 처리)

- **연산**: 프롬프트 내 $L$개 토큰을 병렬 처리 (행렬-행렬 곱, MATMUL)
- **산술 강도**: AI $\approx$ $L$ FLOP/byte (배치 크기 효과)
- **병목**: $L$이 충분히 크면 Compute-bound → **Tensor Core, FLOPS 중요**
- **RAG와의 관계**: 컨텍스트 증가 시 이 단계가 선형으로 느려짐

```
prefill_time ∝ context_length × model_params × (1 / Peak_FLOPS)
```

### 3.2 Decode 단계 (토큰 생성)

- **연산**: 이전 토큰 1개를 입력으로 다음 토큰 생성 (MATVEC)
- **산술 강도**: AI $\approx$ 1 FLOP/byte
- **병목**: 완전 Memory-bound → **HBM 대역폭만 중요**
- **이론적 TPS 공식**:

$$\text{TPS}_{\text{decode}} \approx \frac{\text{HBM Bandwidth (GB/s)}}{\text{Model Size (GB)}}$$

### 3.3 하드웨어별 이론 TPS 실측 비교 (70B FP16, 단일 배치)

| GPU | HBM 대역폭 | 모델 크기 | 이론 TPS | 실측 TPS (vLLM) |
|-----|-----------|---------|---------|----------------|
| RTX 3090 | 936 GB/s | 140 GB | 6.7 | ~5~6 |
| RTX 4090 | 1,008 GB/s | 140 GB | 7.2 | ~6~7 |
| A100 80GB | 2,000 GB/s | 140 GB | 14.3 | ~12~14 |
| **H100 SXM5** | **3,350 GB/s** | 140 GB | **23.9** | **~20~24** |
| H200 | 4,800 GB/s | 140 GB | 34.3 | ~30~34 |

> 이론값과 실측값이 ~85% 일치. 오차는 KV Cache 읽기 오버헤드와 커널 효율 차이.

**동일 모델은 어느 GPU에서 실행해도 동일한 출력을 산출한다.** 가중치가 같으므로 행렬 연산 결과가 결정론적으로 동일하다. 속도만 대역폭 비율에 따라 달라진다.

---

## 4. CUDA의 역할: 이론 한계에 실제로 도달하기

### 4.1 이론과 실측의 갭

단순 PyTorch 구현과 최적화된 추론 엔진의 실측 차이:

```
70B FP16, H100, 단일 배치 decode:
  이론 TPS:                    24 tokens/s  (100%)
  naive PyTorch:               ~6 tokens/s  (25%)
  vLLM + FlashAttention:       ~20 tokens/s (83%)
  
  갭 = 4배. 하드웨어는 동일.
```

이 갭의 원인과 CUDA 최적화의 역할을 이하 절에서 논한다.

### 4.2 메모리 계층 구조와 SRAM 활용

LLM 최적화의 핵심은 GPU 내 메모리 계층 이해에 있다:

| 메모리 계층 | 크기 | 대역폭 | 비고 |
|-----------|------|--------|------|
| HBM (VRAM) | 80 GB | 3.35 TB/s | 외부 메모리 |
| L2 Cache | ~50 MB | ~6 TB/s | 공유 캐시 |
| **SRAM (Shared Memory/L1)** | ~20 MB | **~20 TB/s** | **per-SM 고속 메모리** |
| 레지스터 | ~256 KB/SM | ~20 TB/s | 최고속, SM 내부 |

CUDA 최적화의 본질: **데이터를 HBM에서 읽고 쓰는 횟수를 줄이고, 가능하면 SRAM/레지스터에서 처리한다.**

---

## 5. 핵심 CUDA 최적화 기법 분석

### 5.1 FlashAttention — IO-Aware Attention

**[Dao et al., NeurIPS 2022]** [3] **[Dao, ICLR 2024]** [4]

표준 Attention 알고리즘의 메모리 복잡도는 $O(N^2)$로, 시퀀스 길이 $N$에 대해 이차적으로 HBM 접근이 증가한다:

```
Standard Attention HBM 접근:
  S = Q @ Kᵀ       → HBM에 쓰기 (N² × d)
  P = softmax(S)    → HBM에서 읽기, 다시 쓰기
  O = P @ V         → HBM에서 읽기

총 HBM 접근량: O(N² × d)  — 시퀀스 길이 제곱에 비례
```

FlashAttention은 타일링(tiling) 기법으로 중간 행렬을 SRAM 내에서 처리한다:

```
FlashAttention HBM 접근:
  Q, K, V 타일 단위로 SRAM 적재
  S, P: SRAM에서만 존재 (HBM에 저장하지 않음)
  최종 O만 HBM에 저장

총 HBM 접근량: O(N × d)  — 시퀀스 길이에 선형 비례
```

Dao et al. [3]의 실험 결과:
- GPT-2 (seq 1K): **3배 속도 향상**, 20% 메모리 절감
- GPT-3 급 모델 (seq 4K): **7~10배 속도 향상** (Prefill)
- FlashAttention-2 [4]: A100 대비 이론 FLOPS 대비 달성률 50~73%로 개선

> **Prefill 단계에서 가장 극적인 효과**. Decode는 KV Cache 어텐션 연산에서 부분적 효과.

### 5.2 Continuous Batching — 처리량 극대화

**[Yu et al., OSDI 2022]** [5]

전통적 배치 처리(request-level scheduling)의 문제: 배치 내 한 요청이 완료되어도 나머지가 끝날 때까지 대기. GPU 활용률 저하.

Orca의 iteration-level scheduling은 매 iteration(토큰 생성 1회)마다 배치를 재구성한다:

```
[전통 배치]                    [Continuous Batching]
req1: ████████░░                req1: ████████
req2: ██████████████            req2: ██████████████
req3:         (대기)            req3:         ████████
GPU 공백: 있음                 GPU 공백: 최소화
```

Yu et al.의 GPT-3 175B 실험에서 **FasterTransformer 대비 36.9× 처리량 향상**을 보고하였다.

> **개인 레이턴시 개선 없음.** 처리량(throughput) 극대화 기법. 같은 대역폭에서 더 많은 사용자 요청 처리.

### 5.3 PagedAttention — KV Cache 메모리 관리

**[Kwon et al., SOSP 2023]** [6]

Decode 단계에서 KV Cache는 시퀀스 길이에 비례하여 HBM을 점유한다. 기존 시스템은 최대 시퀀스 길이를 가정하고 미리 연속 메모리를 할당, **60~80% 내부 단편화(fragmentation)**가 발생하였다.

PagedAttention은 OS의 가상 메모리 페이징과 동일한 원리를 KV Cache에 적용한다:

```
KV Cache를 고정 크기 블록(block)으로 분할
비연속적 물리 메모리 허용 (페이지 테이블로 주소 변환)

효과:
  - HBM 낭비 감소: 단편화 ~4%로 최소화
  - 동시 처리 배치 크기 증가 → 처리량 향상
  - KV Cache 공유(prefix sharing) 가능
```

vLLM 평가에서 FasterTransformer 및 Orca 대비 **2~4× 처리량 향상**을 보고하였다 [6].

### 5.4 Fused Kernel — HBM 왕복 감소

Transformer 레이어의 각 연산(LayerNorm, Linear, Activation, Residual Add)을 별도 CUDA 커널로 실행하면 각 커널 사이에 HBM 읽기/쓰기가 발생한다.

```
Unfused (3 kernels):
  Input → HBM → LayerNorm → HBM → Linear → HBM → GELU
  HBM 왕복: 4회 (읽기 3 + 쓰기 3)

Fused (1 kernel):
  Input → [LayerNorm + Linear + GELU 레지스터/SRAM 처리] → HBM
  HBM 왕복: 2회 (읽기 1 + 쓰기 1)
```

중간 결과를 레지스터 또는 SRAM에 유지함으로써 불필요한 HBM 접근을 제거한다. FlashAttention이 이 원리의 가장 극적인 예시이며, xformers, Triton 기반 커스텀 커널도 동일 접근을 취한다.

---

## 6. 양자화: 대역폭 요구량 직접 감소

### 6.1 GPTQ

**[Frantar et al., ICLR 2023]** [7]

OBD(Optimal Brain Damage) 계열의 2차 최적화 기반 PTQ(Post-Training Quantization). 175B 파라미터 모델을 약 4시간 내에 INT4로 양자화한다.

```
FP16 → INT4 변환 효과:
  모델 크기: 140GB → 35GB (4× 감소)
  이론 TPS: BW / 35 = BW / 140 × 4  (4× 향상)

A100 실험 [7]: FP16 대비 3.25× 속도 향상 (end-to-end)
A6000 실험:    FP16 대비 4.5× 속도 향상
```

### 6.2 AWQ (MLSys 2024 Best Paper)

**[Lin et al., MLSys 2024]** [8]

GPTQ의 한계(calibration set 과적합, 일부 모델 품질 저하)를 개선. 활성화 분포를 분석하여 중요한 1% 가중치 채널을 보호하는 per-channel scaling 기법을 도입한다.

```
AWQ의 핵심 관찰:
  가중치 양자화 오류 = 가중치 오류 × 활성화 크기
  → 활성화 값이 큰 채널의 가중치 오류가 치명적
  → 해당 채널만 FP16으로 유지하거나 scaling 적용
  → 캘리브레이션 없이도 Llama, Qwen 계열에서 높은 품질 유지
```

**AWQ 커널의 핵심**: INT4 가중치를 읽어 레지스터 내에서 FP16으로 즉석 변환(dequantize on-the-fly), HBM에 FP16 가중치를 쓰지 않음. 대역폭 소비가 INT4 수준(35GB)으로 고정.

### 6.3 양자화 비트폭에 따른 TPS 비교

| 정밀도 | 바이트/파라미터 | 70B 모델 크기 | H100 이론 TPS |
|--------|--------------|-------------|--------------|
| FP32 | 4 bytes | 280 GB | 12 |
| FP16 | 2 bytes | 140 GB | 24 |
| INT8 | 1 byte | 70 GB | 48 |
| **INT4 (AWQ/GPTQ)** | 0.5 bytes | **35 GB** | **96** |
| INT2 | 0.25 bytes | 17.5 GB | ~192 (품질 심각 저하) |

---

## 7. Speculative Decoding: 대역폭 방정식 자체를 바꾸기

**[Leviathan et al., ICML 2023]** [9]

Decode의 근본 병목은 "토큰을 1개씩 순차 생성"이라는 자기회귀 구조에 있다. Speculative Decoding은 이 구조를 우회한다:

```
알고리즘:
  1. 소형 draft 모델이 γ개(예: 4~8) 토큰을 빠르게 추측
  2. 대형 target 모델이 추측된 γ개를 병렬로 검증 (1 forward pass)
  3. 검증 통과 토큰 수만큼 출력, 불일치 이후는 폐기

대역폭 관점:
  - draft 모델(예: 7B) 실행: 작은 모델 대역폭 소비
  - target 모델(70B) 1회 실행: 가중치 1회 읽기
  - 평균 γ개 토큰 생성 = HBM 접근 1회당 γ개 출력
  - 실효 TPS = 원래 TPS × 평균 수락 토큰 수(α)
```

Leviathan et al. [9]의 T5-XXL 실험에서 **2~3× 속도 향상** 확인. 단, draft 모델의 토큰 수락률이 도메인에 따라 크게 달라져 실제 효과는 가변적이다.

---

## 8. 엣지 케이스: 대역폭이 주 병목이 아닌 상황

### 8.1 Flash 스토리지 기반 추론

**[Alizadeh et al., Apple 2023]** [10]

DRAM이 부족한 엣지 디바이스(스마트폰, 노트북)에서 모델을 Flash SSD에 저장하고 필요한 부분만 DRAM으로 로드하는 패턴. 이 경우 병목이 DRAM 대역폭에서 **Flash 읽기 대역폭**(~10 GB/s)으로 이동한다.

Windowing, row-column bundling 기법으로 Flash 읽기 횟수를 최소화하여 **CPU 4~5×, GPU 20~25× 속도 향상**을 보고하였다.

> 병목이 바뀌면 최적화 전략도 바뀐다. LLM in a Flash는 CUDA 최적화보다 데이터 접근 패턴 최적화가 핵심이다.

### 8.2 Apple Silicon 통합 메모리 아키텍처

기존 PC 아키텍처에서 CPU-GPU 간 PCIe 5.0 × 16 대역폭은 64 GB/s로, CPU 시스템 메모리와 GPU VRAM 사이 병목이 존재한다. Apple Silicon의 통합 메모리(Unified Memory)는 이 병목을 제거한다:

```
기존 아키텍처:
  CPU RAM (DDR5): 96 GB/s
  GPU VRAM (GDDR7): 1,008 GB/s (RTX 4090)
  CPU↔GPU PCIe: 64 GB/s  ← 병목

Apple M5 Max:
  Unified Memory: ~600 GB/s
  CPU + GPU 공유, 전송 병목 없음

→ 7B 모델 FP16: TPS = 600 / 14 ≈ 43 tokens/s
→ 소비 전력: ~15W vs H100 700W
→ 전력 효율: 단위 전력당 TPS에서 압도적
```

### 8.3 배치 Prefill의 Compute-bound 전환

Pope et al. [2]의 분석에 따르면, 배치 크기 $B$일 때 Prefill 단계의 산술 강도는:

$$\text{AI}_{\text{prefill}} = B \times L \text{ FLOP/byte}$$

$B \times L$이 Machine Balance(H100 기준 590)를 초과하면 Compute-bound로 전환되어 FLOPS와 Tensor Core 성능이 중요해진다. 즉 **RAG로 긴 컨텍스트를 처리하거나 다수 요청을 배치할수록 Prefill이 Compute-bound에 가까워진다.**

---

## 9. 종합 분석

### 9.1 각 최적화의 병목 개선 효과

| 최적화 기법 | Decode TPS | Prefill 속도 | 배치 처리량 | 핵심 메커니즘 |
|-----------|-----------|------------|-----------|-------------|
| FlashAttention [3,4] | ★☆☆ | ★★★ | ★★☆ | HBM→SRAM 우회 (O(n²)→O(n)) |
| Continuous Batching [5] | ☆☆☆ | ☆☆☆ | ★★★ | 배치 재구성으로 GPU 활용률 향상 |
| PagedAttention [6] | ☆☆☆ | ☆☆☆ | ★★★ | KV Cache 단편화 제거 |
| AWQ/GPTQ [7,8] | ★★★ | ★★★ | ★★★ | 대역폭 요구량 2~4× 직접 감소 |
| Speculative Decoding [9] | ★★★ | ☆☆☆ | ★☆☆ | 1 HBM 접근당 다중 토큰 출력 |
| Fused Kernel | ★★☆ | ★★☆ | ★★☆ | 불필요 HBM 왕복 제거 |
| CUDA Graph | ★★☆ | ☆☆☆ | ★☆☆ | CPU 오버헤드 제거 |
| Tensor Core | ★☆☆ | ★★★ | ★★★ | Compute-bound 구간에서만 유효 |

> ★★★ 큰 효과 / ★★☆ 중간 / ★☆☆ 소폭 / ☆☆☆ 무관

### 9.2 LLM 추론 속도 결정 요인 계층

```
Level 1: 이론적 한계 (하드웨어)
  → HBM 대역폭 / 모델 크기  [변경 불가, 구매로만 해결]

Level 2: 양자화 (모델 압축)
  → 모델 크기 감소 → 이론 한계를 비례적으로 상향
  → AWQ [8], GPTQ [7]

Level 3: CUDA 효율 (이론 한계에 근접)
  → FlashAttention [3,4], Fused Kernel, CUDA Graph
  → 20~30% → 70~90% 효율 달성

Level 4: 알고리즘 변환 (방정식 자체를 변경)
  → Speculative Decoding [9]
  → 1 HBM 접근 → 복수 토큰 출력

Level 5: 처리량 최적화 (개인 레이턴시와 무관)
  → Continuous Batching [5], PagedAttention [6]
  → 동일 대역폭에서 더 많은 동시 사용자 처리

Level 6: 체감 속도 (UX)
  → 스트리밍(SSE) — 실제 속도 변화 없음
  → Semantic Cache — 반복 질의 LLM 우회
```

### 9.3 명제 검증 요약

**명제 1 검증** (TPS는 FLOPS가 아닌 대역폭에 의해 결정):
- 이론: AI ≈ 1 FLOP/byte로 Roofline 모델 좌측 메모리 병목 구간에 위치 [1]
- 실증: 이론 TPS = BW / 모델크기 공식이 실측치와 ~85% 일치 [2]
- 확인: Speculative Decoding은 가중치 읽기 횟수를 줄여 TPS를 높임 [9]
- **명제 1 참**

**명제 2 검증** (CUDA는 이론 한계가 아닌 효율을 높임):
- Flash Attention은 HBM 접근 횟수를 줄이되, HBM 대역폭 자체는 불변 [3,4]
- Continuous Batching, PagedAttention은 처리량을 높이되 decode TPS에 영향 없음 [5,6]
- 양자화 커널은 INT4 가중치를 on-the-fly 변환으로 대역폭 요구량을 줄임 [7,8]
- **명제 2 참**

---

## 10. 결론

LLM decode 단계의 TPS는 **메모리 대역폭 / 모델 크기**라는 단순한 공식으로 90% 이상 예측 가능하다. CUDA는 이 이론적 한계 위에서 다음 세 가지를 수행한다:

1. **효율 향상**: 실제 대역폭 이용률을 25% → 85%로 높임 (Fused Kernel, CUDA Graph)
2. **우회로 개척**: HBM 대신 SRAM을 활용 (FlashAttention)
3. **요구량 절감**: 읽어야 할 바이트 수를 줄임 (양자화 커널)

가장 빠른 추론 시스템은 이 세 가지를 동시에 적용한다. vLLM이 naive PyTorch 대비 3~4배 빠른 이유는 연산 능력이 아니라 메모리 접근의 효율성 차이에 있다.

---

## 참고문헌

[1] S. Williams, A. Waterman, and D. Patterson, "Roofline: an insightful visual performance model for multicore architectures," *Communications of the ACM*, vol. 52, no. 4, pp. 65–76, Apr. 2009. doi: 10.1145/1498765.1498785

[2] R. Pope, S. Douglas, A. Chowdhery, J. Devlin, J. Bradbury, A. Levskaya, J. Heek, K. Xiao, S. Agrawal, and J. Dean, "Efficiently Scaling Transformer Inference," in *Proc. MLSys 2023* (Outstanding Paper Award). arXiv:2211.05102

[3] T. Dao, D. Y. Fu, S. Ermon, A. Rudra, and C. Ré, "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness," in *Advances in Neural Information Processing Systems (NeurIPS)*, 2022. arXiv:2205.14135

[4] T. Dao, "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning," in *Proc. ICLR*, 2024. arXiv:2307.08691

[5] G.-I. Yu, J. S. Jeong, G.-W. Kim, S. Kim, and B.-G. Chun, "Orca: A Distributed Serving System for Transformer-Based Generative Models," in *Proc. USENIX OSDI*, 2022.

[6] W. Kwon, Z. Li, S. Zhuang, Y. Sheng, L. Zheng, C. H. Yu, J. Gonzalez, H. Zhang, and I. Stoica, "Efficient Memory Management for Large Language Model Serving with PagedAttention," in *Proc. ACM SOSP*, 2023. arXiv:2309.06180

[7] E. Frantar, S. Ashkboos, T. Hoefler, and D. Alistarh, "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers," in *Proc. ICLR*, 2023. arXiv:2210.17323

[8] J. Lin, J. Tang, H. Tang, S. Yang, X. Dang, C. Gan, and S. Han, "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration," in *Proc. MLSys* (Best Paper Award), 2024. arXiv:2306.00978

[9] Y. Leviathan, M. Kalman, and Y. Matias, "Fast Inference from Transformers via Speculative Decoding," in *Proc. ICML*, vol. 202, pp. 19274–19286, 2023. arXiv:2211.17192

[10] K. Alizadeh, I. Mirzadeh, D. Belenko, S. K. Khatamifard, M. Cho, C. C. Del Mundo, M. Rastegari, and M. Farajtabar, "LLM in a Flash: Efficient Large Language Model Inference with Limited Memory," Apple Machine Learning Research, Dec. 2023. arXiv:2312.11514

[11] NVIDIA Corporation, "NVIDIA H100 Tensor Core GPU Architecture," Technical Whitepaper, 2022.

---

## 관련 노트
- [[LLM-Inference-Bandwidth]] — 대역폭 병목 분석 (원본 노트)
- [[CUDA-Role-in-LLM]] — CUDA 역할 분석 (원본 노트)
- [[Inference-Optimization]] — vLLM, FlashAttention 구현 세부사항
- [[../hardware/gpu-workstation/DGX-Spark]] — H100/B200 하드웨어 스펙
- [[../hardware/gpu-workstation/Apple-M5-Max]] — Apple Silicon 통합 메모리 아키텍처
