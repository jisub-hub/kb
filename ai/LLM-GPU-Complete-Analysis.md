---
tags:
  - ai
  - research
  - report
  - cuda
  - inference
  - training
  - bandwidth
  - tps
  - hardware
  - throughput
created: 2026-06-16
---

# LLM GPU 성능 완전 분석 (v2)
## — 추론 속도 · 다중사용자 처리량 · 학습 하드웨어 통합 가이드

> **목적**: "TPS는 대역폭이다", "CUDA는 무슨 역할인가", "동시 사용자 처리량은 어떻게 계산하나",  
> "CUDA 성능 높은 카드가 학습에 강한가" — 이 네 가지 질문을 하나의 일관된 프레임으로 분석한다.

---

## 0. 한 눈에 요약

```
질문                           답
─────────────────────────────────────────────────────────────────
TPS를 결정하는 것은?           HBM 대역폭 / 모델 크기  (decode 기준)
CUDA는 무슨 역할인가?          이론 한계(대역폭)에 실제로 도달하게 함
다수 사용자 처리량은?           배칭으로 가중치 읽기를 공유 → N배 처리량
                               단, KV Cache VRAM이 사용자 수를 제한 (GQA면 8× 완화)
학습에 강한 카드는?             Dense Tensor Core TFLOPS 높은 카드 (H100, A100)
추론에 강한 카드는?             HBM 대역폭 높고 VRAM 큰 카드 (H200, B200)
"CUDA 성능 높은" 게임카드는?    FP32 셰이더가 많은 것 → ML에서 덜 중요
─────────────────────────────────────────────────────────────────
```

---

## 1. 이론적 기반: Roofline 모델

### 1.1 산술 강도 (Arithmetic Intensity)

모든 분석의 출발점은 **산술 강도(AI)**다 [Williams et al., CACM 2009]:

```
AI = 수행한 FLOP 수
     ─────────────────
     이동한 데이터 (Bytes)

단위: FLOP/byte
```

하드웨어의 **기계적 균형점(Machine Balance)**:

```
Machine Balance = Peak FLOPS / Peak BW

H100 SXM5:  1,979 TFLOPS / 3.35 TB/s  = 590 FLOP/byte
A100 80GB:    312 TFLOPS / 2.00 TB/s  = 156 FLOP/byte
RTX 4090:     165 TFLOPS / 1.01 TB/s  = 163 FLOP/byte  (Tensor Core FP16 기준)
```

AI < Machine Balance → Memory-bound (대역폭이 병목)
AI > Machine Balance → Compute-bound (FLOPS가 병목)

> [!warning] 스펙시트 함정 — 2:4 구조적 희소성 (Structured Sparsity)
> NVIDIA가 공개하는 H100 SXM5의 **1,979 TFLOPS**는 **2:4 Structured Sparsity** 적용 시의 수치다.
> 4개 가중치 중 2개를 강제로 0으로 만들어 하드웨어가 절반만 실제 연산하도록 하는 방식이다.
> 일반 Dense LLM 가중치에서의 베이스라인은 그 절반인 **989.5 TFLOPS**다.
>
> ```
> 실질 Machine Balance 비교:
>   H100 SXM5 (Dense, 실제): 989.5 TFLOPS / 3.35 TB/s = 295 FLOP/byte
>   H100 SXM5 (Sparse, 마케팅): 1,979 TFLOPS / 3.35 TB/s = 590 FLOP/byte
> ```
>
> Decode(AI ≈ 1)는 어느 수치를 써도 완전 Memory-bound라는 결론이 바뀌지 않는다.
> 그러나 **Prefill이 Compute-bound로 전환되는 임계 토큰 수**와 **학습 속도 비교** 시에는
> Dense 수치(995 TFLOPS)를 기준으로 삼아야 한다.

---

## 2. LLM 추론의 두 단계

LLM inference는 연산 구조가 다른 두 단계로 구성된다 [Pope et al., MLSys 2023]:

### 2.1 Prefill 단계 (프롬프트 처리)

```
연산:      행렬-행렬 곱 (MATMUL)
           — 프롬프트 L개 토큰을 병렬 처리
AI:        ≈ L FLOP/byte  (L = 토큰 수)
병목:      L이 크면 Compute-bound
           → Tensor Core, FLOPS가 중요

실측 (H100, L=2048 프롬프트):
  prefill 시간 ≈ L × model_params × 2 / Peak_FLOPS
              ≈ 2048 × 70B × 2 / 1,979T ≈ 0.14초
```

### 2.2 Decode 단계 (토큰 생성)

```
연산:      행렬-벡터 곱 (MATVEC)
           — 직전 토큰 1개로 다음 토큰 예측
AI:        ≈ 1 FLOP/byte  (H100 Dense Machine Balance 295의 1/295)
병목:      완전 Memory-bound
           → HBM 대역폭만 중요

TPS 공식:
  TPS_decode ≈ HBM_BW (GB/s) / Model_Size (GB)
```

### 2.3 하드웨어별 이론 TPS 한계 및 VRAM 적재 가능성 (70B FP16)

> [!warning] 표를 읽기 전에 — 70B FP16은 약 140GB다
> 아래 "이론 TPS"는 **순수 대역폭 공식을 적용한 이론적 상한선**일 뿐이다.
> 70B FP16 모델(≈140GB)은 80GB 이하 단일 GPU에 **올라가지 않아 OOM이 발생**한다.
> 실서비스하려면 **INT4 양자화(≈35GB)** 하거나 **Tensor Parallelism(TP=2 이상)**으로 묶어야 한다.

| GPU | HBM BW | VRAM | 70B FP16 단일 적재 | 이론 TPS 상한 |
|-----|--------|------|------------------|-------------|
| RTX 3090 | 936 GB/s | 24 GB | 불가 (TP=6+ 필요) | ~6.7 |
| RTX 4090 | 1,008 GB/s | 24 GB | 불가 (TP=6+ 필요) | ~7.2 |
| A100 80GB | 2,000 GB/s | 80 GB | 불가 (TP=2 필요) | ~14.3 |
| **H100 SXM5** | **3,350 GB/s** | 80 GB | 불가 (TP=2 필요) | **~23.9** |
| H200 | 4,800 GB/s | 141 GB | 가중치만 겨우 (실서비스 불가) | ~34.3 |

> 이론값과 실측값이 ~85% 일치. 오차는 KV Cache 읽기 오버헤드.
> **동일 모델은 어느 GPU에서도 동일한 결과를 출력**한다. 속도만 대역폭 비율에 따라 다르다.

### 2.4 특수 케이스: MoE (Mixture of Experts) 모델

MoE 모델(Mixtral 8x7B, DeepSeek-V3 등)은 기존 Dense 모델과 다른 Roofline 프로필을 가진다.

```
Mixtral 8x7B 구조:
  총 파라미터: 47B  → VRAM 94 GB (FP16) 필요
  활성 파라미터: 13B (매 토큰마다 8개 Expert 중 2개만 활성화)

Dense 모델 공식 적용 시 오류:
  TPS = BW / model_size = BW / 94 GB  ← 틀림

MoE 올바른 공식:
  TPS = BW / active_params_size = BW / 26 GB  (FP16 기준)
  H100: 3,350 / 26 ≈ 129 TPS  (47B Dense 모델로 오해하면 36 TPS)
```

MoE 모델의 실제 특성:

| 항목 | Dense 70B | MoE 47B (Mixtral) |
|------|----------|------------------|
| 필요 VRAM | 140 GB | 94 GB |
| 활성 BW 소비 | 140 GB/token | 26 GB/token |
| 이론 TPS (H100) | 24 | ~129 |
| 품질 수준 | 70B급 | 13B~70B 사이 |

> MoE는 "VRAM 요구량 = 전체 파라미터, 속도 = 활성 파라미터"라는 비대칭 구조를 가진다.
> VRAM이 충분하다면 동일 품질 대비 압도적으로 빠르다.
> Expert 라우팅 불균형(load imbalance) 문제가 실측 TPS를 낮출 수 있다.

---

## 3. CUDA의 역할: 이론 한계에 도달하기

### 3.1 이론과 실측의 갭

```
70B FP16, H100, 단일 사용자 decode:

  이론 TPS:                     24 tokens/s  (100%)
  naive PyTorch:                ~6 tokens/s  ( 25%)
  vLLM + FlashAttention:       ~20 tokens/s  ( 83%)

  차이: 3.3배. 하드웨어는 동일하다.
```

이 갭의 원인: GPU 내부 메모리 계층을 비효율적으로 사용하기 때문이다.

```
GPU 메모리 계층 (H100 기준):
  레지스터  ~256 KB/SM  ~20 TB/s   최고속, SM 내부
  SRAM/L1   ~20 MB      ~20 TB/s   HBM의 6배 속도
  L2 Cache  ~50 MB      ~6 TB/s
  HBM       ~80 GB      ~3.35 TB/s ← "대역폭"이라 말하는 그것
```

CUDA 최적화의 본질: **HBM 접근 횟수를 줄이고, 연산을 SRAM/레지스터에서 처리한다.**

### 3.2 FlashAttention — HBM 왕복 O(n²) → O(n)

표준 Attention은 중간 행렬(S, P)을 HBM에 저장한다 [Dao et al., NeurIPS 2022]:

```
Standard Attention:
  S = Q @ Kᵀ   → HBM에 쓰기 (seq_len² 크기)
  P = softmax(S) → HBM에서 읽어 계산 → HBM에 쓰기
  O = P @ V    → HBM에서 읽어 계산
  HBM 접근: O(seq_len²)

FlashAttention (SRAM 타일링):
  Q, K, V를 블록 단위로 SRAM에 적재
  S, P는 SRAM에서만 존재 — HBM에 절대 쓰지 않음
  최종 O만 HBM에 쓰기
  HBM 접근: O(seq_len)  ← 선형으로 감소
```

FlashAttention-2 [Dao, ICLR 2024]: A100에서 이론 FLOP/s 대비 **50~73% 달성률** (기존 20~30% 대비).

### 3.3 Fused Kernel — HBM 왕복 제거

```
Unfused (LayerNorm → Linear → GELU 각각 별도 커널):
  Input → HBM → LayerNorm → HBM → Linear → HBM → GELU → HBM
  HBM 왕복: 6회

Fused (단일 커널):
  Input → [LayerNorm + Linear + GELU, 레지스터에서 처리] → HBM
  HBM 왕복: 2회  →  3배 대역폭 절약
```

### 3.4 양자화 커널 — 대역폭 요구량 직접 감소

INT4 양자화 커널(AWQ [Lin et al., MLSys 2024 Best Paper], GPTQ [Frantar et al., ICLR 2023]):

```
잘못된 구현:
  HBM에서 INT4 읽기 → INT4를 FP16으로 변환 → FP16을 HBM에 저장 → FP16 연산
  (FP16 대역폭 소비, 양자화 효과 없음)

올바른 구현:
  HBM에서 INT4 읽기 (모델 크기 4배 감소)
  → 레지스터 내에서 즉석 FP16 변환 (HBM 쓰기 없음)
  → Tensor Core에서 FP16 연산
  HBM 소비: INT4 크기(35GB) 기준
```

양자화 비트폭별 이론 TPS (H100, 70B 모델):

| 정밀도 | 모델 크기 | 이론 TPS | 실측 배율 |
|--------|---------|---------|---------|
| FP16 | 140 GB | 24 | ×1 |
| INT8 | 70 GB | 48 | ×2 |
| **INT4 (AWQ)** | **35 GB** | **96** | **×4** |

GPTQ 실험: A100에서 FP16 대비 3.25× 속도 향상, A6000에서 4.5× 향상 [Frantar et al.].

### 3.5 CUDA Graph — CPU 오버헤드 제거

```
Decode 루프 (CUDA Graph 없음):
  토큰마다 수백 개 커널 개별 런치 → CPU→GPU 명령 전달 ~1ms/token
  500토큰 = 500ms 순수 오버헤드

CUDA Graph:
  1회 실행 흐름 캡처 → 이후 "그래프 실행" 1번 = ~0.05ms/token
  500토큰 = 25ms  (20배 감소)

효과: 소형 모델(7B, 13B)에서 더 큼 — 모델이 빠를수록 오버헤드 비율이 높아짐
```

### 3.6 최적화별 효과 정리

| 최적화 | Decode TPS | Prefill 속도 | 배치 처리량 | 핵심 메커니즘 |
|--------|-----------|------------|-----------|-------------|
| FlashAttention | ★☆☆ | ★★★ | ★★☆ | HBM→SRAM 우회 |
| Fused Kernel | ★★☆ | ★★☆ | ★★☆ | HBM 왕복 제거 |
| AWQ/GPTQ | ★★★ | ★★★ | ★★★ | 대역폭 요구량 감소 |
| CUDA Graph | ★★☆ | ☆☆☆ | ★☆☆ | CPU 오버헤드 제거 |
| Continuous Batching | ☆☆☆ | ☆☆☆ | ★★★ | GPU 가동률 극대화 |
| PagedAttention | ☆☆☆ | ☆☆☆ | ★★★ | KV Cache 단편화 제거 |
| Speculative Decoding | ★★★ | ☆☆☆ | ★☆☆ | 1 HBM 접근당 다중 토큰 |

---

## 4. 다중 사용자 처리량 분석

### 4.1 배칭의 핵심 원리: 가중치 공유

```
사용자 1명 decode 1 step:
  가중치 읽기: 140 GB (FP16, 70B)
  생성 토큰:  1개

N명 배치 decode 1 step:
  가중치 읽기: 140 GB  ← 동일! 모두가 공유
  생성 토큰:  N개

→ 같은 대역폭 소비로 N배 토큰 생성
→ 단, KV Cache는 사용자마다 별도로 VRAM 점유
```

### 4.2 KV Cache 메모리 구조 — MHA vs GQA가 결정적

> [!important] 핵심: KV head 수가 MHA냐 GQA냐에 따라 8배 차이가 난다
> KV Cache 공식의 head 수는 **attention head가 아니라 KV head(n_kv_heads)**다.
> 구형 MHA는 KV head = attention head(64)지만, Llama 3 등 최신 GQA는 KV head=8뿐이다.

```
KV Cache 공식:
  KV_bytes/token = 2(K+V) × layers × n_kv_heads × head_dim × bytes

구형 70B (MHA): n_kv_heads = 64 (= attention head)
  = 2 × 80 × 64 × 128 × 2 bytes ≈ 2.62 MB / token

Llama 3 70B (GQA): n_kv_heads = 8 (attention head는 64지만 KV는 8개로 공유)
  = 2 × 80 × 8 × 128 × 2 bytes = 327,680 bytes ≈ 0.33 MB / token  ← 1/8
```

**결론: GQA 덕분에 Llama 3 70B의 KV Cache는 구세대(MHA) 대비 1/8로 감소했다.**
동일 VRAM에서 약 8배 많은 동시 사용자를 수용할 수 있다. 이하 모든 계산은 **Llama 3 GQA(0.33 MB/token, FP16)** 기준이다.

```
GQA 기준 (FP16 KV 0.33 MB/token), context 2048:
  사용자 1명: 0.33 MB × 2048 ≈ 0.67 GB
  사용자 32명: 0.67 GB × 32 = 21.6 GB
  최대 동시 사용자: 45 GB 여유 / 0.67 GB ≈ 66명
  (80GB - 35GB INT4 모델 = 45GB 여유)
  → 구형 MHA였다면 같은 조건에서 ~8명에 그쳤을 것
```

### 4.2.1 KV Cache 양자화 — 동시 사용자 수를 2~4× 확장

가중치 양자화(AWQ/GPTQ)가 모델 크기를 줄이듯, **KV Cache 자체도 FP8/INT4로 압축**할 수 있다.
vLLM 0.4+, SGLang 등에서 표준 기능으로 지원된다.

```
KV Cache 정밀도별 비교 (Llama 3 70B GQA, context 2048, H100 80GB INT4 모델):
* 45GB 여유 기준. GQA FP16 KV = 0.33 MB/token

정밀도  kv/token   최대 동시 사용자  정확도 영향
────────────────────────────────────────────────────────
FP16    0.33 MB    66명             기준
FP8     0.16 MB    132명 (+2×)      MMLU 차이 <0.3% (프로덕션 안전)
INT4    0.08 MB    264명 (+4×)      ~1~2% 차이 (도메인별 허용 여부 검토)
────────────────────────────────────────────────────────
```

KV Cache 양자화의 핵심 메커니즘:

```
FP8 KV Cache (예: vLLM fp8 kv-cache):
  Attention score 계산 전 K/V를 FP8으로 양자화하여 VRAM에 저장
  Attention 연산 직전 FP16으로 역변환 (레지스터에서)
  → 저장 크기 2×, 대역폭 소비 2× 감소
  → FP16 KV와 수치적으로 거의 동일
```

> 다중 사용자 서비스에서 **가중치 INT4 양자화** 다음으로 적용해야 할 VRAM 최적화다.
> 두 가지를 함께 쓰면 FP16 기준 대비 동시 사용자 8× 이상 수용이 이론적으로 가능하다.

### 4.3 다중 사용자 처리량 공식

decode 1 step에서 읽어야 하는 바이트 = 모델 가중치(1회) + 모든 사용자의 누적 KV Cache.
누적 KV는 context length에 비례하므로, 차원을 보정한 1차 근사 공식은:

```
총 TPS ≈ N × BW
         ──────────────────────────────────────────
         model_size + N × context_len × kv_per_token

BW            = HBM 대역폭
model_size    = 모델 가중치 바이트 (INT4 70B: 35 GB)
context_len   = 사용자당 누적 컨텍스트 길이 (토큰)
kv_per_token  = KV Cache 바이트/token/위치 (Llama 3 GQA FP16: 0.33 MB)
```

**N이 작을 때** (model_size >> N × context_len × kv):
```
총 TPS ≈ N × BW / model_size  →  N에 선형 증가
사용자당 TPS ≈ 단일 사용자와 동일
```

**N이 커질 때** (N × context_len × kv >> model_size):
```
총 TPS ≈ BW / (context_len × kv_per_token)  →  포화
사용자당 TPS ∝ 1/N  →  점점 느려짐
```

### 4.4 실측 근사 수치 (H100 80GB, 70B INT4, Llama 3 GQA, context 2048)

```
조건: INT4 모델 35 GB, HBM BW 3,350 GB/s
      Llama 3 GQA FP16 KV = 0.33 MB/token → 사용자당 2048토큰 = 0.67 GB

배치(N)  모델   KV누적    총VRAM    총 TPS   사용자당 TPS
──────────────────────────────────────────────────────────
 1      35 GB  0.67 GB   35.67 GB    94        94
 8      35 GB  5.40 GB   40.40 GB   663        83
32      35 GB  21.6 GB   56.60 GB  1,894       59
64      35 GB  43.2 GB   78.20 GB  2,743       43
──────────────────────────────────────────────────────────
계산 검증 (N=32):
  32 × 3,350 / (35 + 21.6) = 107,200 / 56.6 = 1,894 ✓

VRAM 한계:
  가용 = 80 - 35 = 45 GB
  최대 동시 사용자 = 45 GB / 0.67 GB ≈ 66명
  (구형 MHA 구조였다면 8명도 어려웠을 것)
```

### 4.5 배칭이 Compute-bound로 전환되는 조건

Roofline 모델에서 배치 크기 N일 때:

```
AI_decode = N FLOP/byte

H100 Dense Machine Balance = 295 FLOP/byte  (Sparse 기준이면 590)
→ N = 295가 되어야 Compute-bound 전환

batch=295, context=2048 (GQA FP16 KV 0.33 MB):
  KV Cache = 295 × 2048 × 0.33 MB ≈ 199 GB → H100 3대 이상 필요

→ 현실에서 LLM decode는 항상 Memory-bound
→ 대규모 배치 prefill만 Compute-bound에 근접
```

### 4.6 Continuous Batching과 PagedAttention

**Static Batching (구형):**
```
[Req1 ████████░░░]  ← 완료 후 GPU가 대기
[Req2 ██████████████]
[Req3 (대기)]
GPU 공백 발생 → 처리량 낭비
```

**Continuous Batching [Yu et al., OSDI 2022] (Orca):**
```
매 iteration마다 완료된 슬롯에 새 요청 삽입
GPU 가동률 지속 유지
→ FasterTransformer 대비 36.9× 처리량 향상
```

**PagedAttention [Kwon et al., SOSP 2023] (vLLM):**
```
기존: 최대 context 길이만큼 VRAM 선점 → 60~80% 단편화
PagedAttention: OS 페이징과 동일 원리, 비연속 물리 메모리 허용
→ 단편화 ~4%로 최소화
→ 동시 처리 배치 크기 증가 → 2~4× 처리량 향상
```

---

## 5. 학습 vs 추론: 하드웨어 최적화의 분기점

### 5.1 연산 구조 비교

```
학습 (Training):
  Forward pass:  배치 × 시퀀스 행렬-행렬 곱 (MATMUL)
  Backward pass: 그래디언트 계산 = 마찬가지로 MATMUL
  배치 크기:     64~512 sequences × 2048 tokens
  산술 강도:     수천 FLOP/byte  → Compute-bound
  병목:          Tensor Core TFLOPS

추론 decode (Inference):
  Autoregressive MATVEC, 배치=1~수십
  산술 강도: 1~수십 FLOP/byte  → Memory-bound
  병목:      HBM 대역폭
```

### 5.2 CUDA Core vs Tensor Core: 마케팅과 실제

```
CUDA Core (FP32 Shader):
  목적: 게임 렌더링, FP32 범용 연산
  ML 기여: 낮음 — training/inference 모두 FP16/BF16 사용

Tensor Core:
  목적: 행렬 곱 가속 (FP16×FP16 → FP32 accumulation)
  ML 기여: 결정적
    Training:   Tensor Core TFLOPS = 학습 속도
    Prefill:    Tensor Core + FlashAttention = 처리 시간
    Decode:     기여 낮음 (Memory-bound이므로)
```

H100의 FP32 CUDA Core 성능은 67 TFLOPS로 RTX 4090(82 TFLOPS)보다 낮다.
그러나 Tensor Core FP16 성능은 1,979 TFLOPS vs 165 TFLOPS — **12배 차이**.

> [!warning] 스펙시트의 1,979 TFLOPS는 Sparse(희소) 기준
> 실제 Dense LLM 학습/추론의 베이스라인은 **989.5 TFLOPS**다 (섹션 1.1 참조).
> A100(312 TFLOPS Dense) 대비 H100은 Dense 기준으로도 **3.2× 향상**이다.
> RTX 4090(165 TFLOPS)과 비교하면 H100 Dense 기준 **6×**, Sparse 기준 **12×**다.

### 5.3 GPU 종합 성능 비교표 (Dense / Sparse 분리)

> 인프라 산정에는 마케팅용 Sparse가 아닌 **Dense 수치**를 써야 한다. Sparse는 2:4 구조적 희소성 적용 시에만 도달한다.

| GPU | FP16 Dense | FP16 Sparse | HBM BW | VRAM | 타겟 용도 |
|-----|-----------|------------|--------|------|----------|
| RTX 4090 | 165 TFLOPS | 330 TFLOPS | 1.01 TB/s | 24 GB GDDR6X | 소규모 튜닝·개인 연구 |
| A100 80GB | 312 TFLOPS | 624 TFLOPS | 2.00 TB/s | 80 GB HBM2e | 학습 + 대형 추론 (가성비) |
| **H100 SXM5** | **989.5 TFLOPS** | **1,979 TFLOPS** | **3.35 TB/s** | **80 GB HBM3** | **대형 학습 + 고성능 추론** |
| H200 SXM | 989.5 TFLOPS | 1,979 TFLOPS | 4.80 TB/s | 141 GB HBM3e | 추론 특화 (최고 BW/VRAM) |
| B200 | ~2,250 TFLOPS | ~4,500 TFLOPS | ~8.0 TB/s | 192 GB HBM3e | 차세대 학습/추론 범용 |
| Apple M5 Max | 공식 미공개 | — | ~0.6 TB/s | 최대 128 GB Unified | 온디바이스 로컬 LLM |

### 5.4 GPU 선택 기준

```
목적별 최적 선택:

대규모 모델 학습 (70B+):
  → H100 SXM5 (Tensor Core TFLOPS + NVLink 집합 대역폭)
  → A100 80GB (보급형)
  핵심 지표: Tensor Core TFLOPS, NVLink 대역폭, VRAM

추론 서비스 (단일 GPU):
  → H100 SXM5 (BW 3,350 GB/s)
  → H200 (BW 4,800 GB/s, 추론에 최적)
  핵심 지표: HBM 대역폭, VRAM

소규모 파인튜닝 (7B~13B LoRA):
  → RTX 4090 (24GB, 가성비)
  핵심 지표: VRAM, Tensor Core TFLOPS

온디바이스 / 저전력:
  → Apple M5 Max (통합 메모리 128GB, ~15W)
  핵심 지표: 통합 메모리 크기, 전력 효율
```

---

## 6. 전체 성능 결정 요인 계층

```
┌─────────────────────────────────────────────────────────┐
│  Level 1: 이론적 한계 (하드웨어)                          │
│    Decode TPS_max = HBM_BW / model_size                  │
│    Training Speed_max = Peak_TFLOPS × MFU               │
│    변경 방법: GPU 업그레이드, 텐서 병렬화(N× BW)           │
├─────────────────────────────────────────────────────────┤
│  Level 2: 양자화 (모델 압축)                              │
│    model_size↓ → Decode TPS↑, VRAM 여유↑                │
│    FP16→INT4: 4× TPS 이론 향상 (AWQ/GPTQ)               │
├─────────────────────────────────────────────────────────┤
│  Level 3: CUDA 효율 (이론 한계에 근접)                    │
│    FlashAttention, Fused Kernel, CUDA Graph              │
│    25% → 83% 대역폭 이용률 달성                           │
├─────────────────────────────────────────────────────────┤
│  Level 4: 알고리즘 변환 (방정식 변경)                     │
│    Speculative Decoding: 1 HBM 접근 → 복수 토큰 출력     │
│    2~3× TPS 향상 (도메인 의존적)                          │
├─────────────────────────────────────────────────────────┤
│  Level 5: 처리량 최적화 (동시 사용자)                     │
│    Continuous Batching (Orca): 36.9× 처리량 향상          │
│    PagedAttention (vLLM): 2~4× 처리량 향상               │
│    개인 레이턴시에는 영향 없음                             │
├─────────────────────────────────────────────────────────┤
│  Level 6: 체감 속도 (UX)                                  │
│    스트리밍(SSE): 첫 토큰 0.3초, 실제 속도 변화 없음       │
│    Semantic Cache: 반복 질의 4000ms → 5ms                 │
└─────────────────────────────────────────────────────────┘
```

---

## 7. 실전 배포 아키텍처 설계 지침

### 7.1 VRAM 계획

```
필요 VRAM = 모델 크기 + KV Cache + 오버헤드

모델 크기:
  FP16: param 수 × 2 bytes
  INT4: param 수 × 0.5 bytes
  70B FP16 = 140 GB, 70B INT4 = 35 GB

KV Cache (FP16):
  = 2 × layers × n_kv_heads × head_dim × 2 bytes × seq_len × batch
  ※ n_kv_heads에 주의 — MHA(64)면 8배 커진다
  Llama 3 70B GQA (n_kv_heads=8), 2048 context, 32 users: ≈ 21.6 GB
  구형 MHA 70B (n_kv_heads=64), 동일 조건:           ≈ 172 GB (적재 불가!)

→ H100 80GB 기준, Llama 3 70B INT4, 32명 동시:
  35 GB (모델) + 21.6 GB (KV) + 2 GB (오버헤드) ≈ 59 GB
  → 여유 있음. 동일 80GB에서 최대 ~66명까지 확장 가능
```

### 7.2 처리량 목표 역산 및 Tensor Parallelism 통신 비용

```
목표: 동시 100명, 사용자당 50 TPS 보장
필요 총 TPS = 5,000 tokens/s

H100 INT4 70B 이론 총 TPS ≈ 2,800 (batch=64 기준)
→ H100 2대 × Tensor Parallelism 필요
→ NVLink 고대역폭 연결 필수 (PCIe는 64 GB/s로 병목)

Tensor Parallelism N대:
  효과적 BW = N × 단일 HBM BW
  2× H100 = 6,700 GB/s 효과적 BW
  70B INT4 TPS ≈ 6,700 / 35 × 배치 효율 ≈ 5,000+ TPS
```

**Tensor Parallelism의 통신 오버헤드 — All-Reduce**

GPU를 N대로 묶으면 각 Transformer 레이어마다 **GPU 간 All-Reduce 통신**이 발생한다.

```
Transformer 1 레이어당 All-Reduce 발생 횟수: 2회
  (Attention 출력 후, FFN 출력 후)

All-Reduce 통신량 (70B, TP=2):
  = 2 × hidden_size × batch × 2bytes
  = 2 × 8192 × 32 × 2 = 1.05 MB / 레이어
  × 80 layers = 84 MB / forward pass

인터커넥트별 통신 시간:
  PCIe 5.0 ×16 (양방향): ~64 GB/s → 84 MB / 64 GB/s = 1.3 ms
  NVLink 4.0 (H100 기준): ~900 GB/s → 84 MB / 900 GB/s = 0.09 ms
  
→ PCIe: 통신 1.3ms vs H100 single decode ~42ms = 오버헤드 3%  (허용 가능)
→ PCIe: 처리량 최대화 배칭 시 통신이 HBM 절약 이점을 잠식
→ NVLink: 오버헤드 0.2% → 선형에 가까운 성능 스케일링
```

**결론: TP로 GPU를 묶을 때 NVLink(또는 InfiniBand)가 없으면 선형 스케일링이 보장되지 않는다.**
대형 클러스터에서 PCIe로 TP=8을 구성하면 통신 오버헤드가 실측 TPS를 이론의 40~60% 수준으로 떨어뜨린다.

### 7.3 비용 효율 최적화

```
1. 가중치 INT4 양자화 (AWQ)
   → TPS 4× 향상 + 동시 사용자 4× 증가
   → 가장 먼저 적용해야 할 최적화, 비용 대비 효과 최고

2. KV Cache FP8 양자화
   → 동시 사용자 추가 2× 확장 (VRAM 절반 소비)
   → 정확도 손실 <0.3%, 프로덕션 안전
   → 가중치 INT4 + KV FP8 병용 시 FP16 대비 ~8× 동시 사용자

3. vLLM + Continuous Batching + PagedAttention
   → 구현 비용 낮음, 처리량 2~4× 즉시 향상
   → 1번, 2번과 독립적으로 중첩 적용 가능

4. Semantic Cache (Redis + 임베딩)
   → 반복 질의 많은 서비스에서 99% 레이턴시 감소
   → LLM 호출 자체를 줄임

5. Speculative Decoding
   → 도메인 특화 draft 모델 필요 (초기 비용 있음)
   → 평균 수락률 높은 도메인에서 2~3× 효과
```

---

## 8. 핵심 명제 검증 요약

| 명제                           | 검증        | 근거                                        |
| ---------------------------- | --------- | ----------------------------------------- |
| TPS는 FLOPS가 아닌 대역폭에 의해 결정된다  | **참**     | AI ≈ 1 FLOP/byte, Roofline 분석             |
| CUDA는 이론 한계가 아닌 효율을 높인다      | **참**     | 25% → 83% 이용률, FlashAttention             |
| 배칭으로 N배 처리량을 얻을 수 있다         | **조건부 참** | KV Cache VRAM이 N을 제한                      |
| CUDA 성능 높은 게임카드는 학습에 강하다     | **일부 참**  | Tensor Core이지 CUDA Core가 아님               |
| H100은 학습과 추론 모두 뛰어나다         | **참**     | Tensor Core 12× + HBM BW 3× (vs 4090)     |
| H100 1,979 TFLOPS는 항상 쓸 수 있다 | **거짓**    | 2:4 Sparsity 적용 시만. Dense 기준 989.5 TFLOPS |
| 70B FP16은 단일 GPU에서 구동 가능하다 | **거짓**    | 140GB 요구. H200이거나 TP=2 또는 양자화 필수         |
| KV Cache는 모델마다 같은 크기다       | **거짓**    | MHA vs GQA가 8배 차이. Llama 3는 GQA로 1/8       |
| MoE 47B는 Dense 47B와 속도가 같다   | **거짓**    | MoE TPS = BW / 활성파라미터. 실제로 ~3× 빠름         |
| TP N대로 묶으면 TPS가 N배가 된다       | **조건부 참** | NVLink 없으면 All-Reduce 오버헤드로 선형 스케일링 불가    |

---

## 9. 참고문헌

[1] S. Williams et al., "Roofline: an insightful visual performance model," *CACM*, 2009. doi:10.1145/1498765.1498785

[2] R. Pope et al., "Efficiently Scaling Transformer Inference," *MLSys 2023* (Outstanding Paper). arXiv:2211.05102

[3] T. Dao et al., "FlashAttention: Fast and Memory-Efficient Exact Attention," *NeurIPS 2022*. arXiv:2205.14135

[4] T. Dao, "FlashAttention-2: Faster Attention with Better Parallelism," *ICLR 2024*. arXiv:2307.08691

[5] G.-I. Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models," *OSDI 2022*.

[6] W. Kwon et al., "Efficient Memory Management for LLM Serving with PagedAttention," *SOSP 2023*. arXiv:2309.06180

[7] E. Frantar et al., "GPTQ: Accurate Post-Training Quantization for GPT," *ICLR 2023*. arXiv:2210.17323

[8] J. Lin et al., "AWQ: Activation-aware Weight Quantization for LLM Compression," *MLSys 2024* (Best Paper). arXiv:2306.00978

[9] Y. Leviathan et al., "Fast Inference from Transformers via Speculative Decoding," *ICML 2023*. arXiv:2211.17192

[10] NVIDIA Corporation, "H100 / H200 / B200 Tensor Core GPU Architecture Whitepapers" (Dense vs Sparse 스펙 분리 적용).

[11] J. Ainslie et al., "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints," *EMNLP 2023*. arXiv:2305.13245

[12] Meta AI, "Llama 3 Model Card," 2024 — Llama 3 70B의 GQA(n_kv_heads=8) 구조 확인.

---

## 개정 이력

**v2 (2026-06-16)** — 외부 리뷰(GPT) 피드백 반영:
- **KV Cache GQA 보정**: Llama 3 70B는 MHA(64 KV head)가 아닌 GQA(8 KV head). token당 2.62 MB → 0.33 MB로 정정 (섹션 4.2)
- **단일 GPU VRAM 현실화**: 70B FP16(140GB)은 80GB 단일 GPU 적재 불가 명시 (섹션 2.3)
- **TPS 공식 차원 보정**: 분모에 context_len 변수 추가 (섹션 4.3)
- **배칭 효율표 재계산**: GQA 기준 N=32에서 총 1,894 TPS, 최대 66명 (섹션 4.4)
- **스펙시트 Dense/Sparse 분리**: H100 989.5(Dense)/1,979(Sparse) TFLOPS 명확화 (섹션 1.2, 5.3)

---

## 관련 노트
- [[LLM-TPS-CUDA-Analysis-Report]] — 학술 논문 인용 중심 상세 보고서
- [[LLM-Inference-Bandwidth]] — TPS 공식 원본 분석
- [[CUDA-Role-in-LLM]] — CUDA 최적화 기법 상세
- [[Inference-Optimization]] — vLLM, FlashAttention 구현
- [[../hardware/gpu-workstation/DGX-Spark]] — H100/B200 하드웨어 스펙
- [[../hardware/gpu-workstation/Apple-M5-Max]] — Apple Silicon 통합 메모리
