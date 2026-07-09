---
tags:
  - ai
  - math
  - matrix
  - floating-point
  - linear-algebra
created: 2026-06-16
---

# Math Foundations for AI

> [!summary] 한 줄 요약
> LLM을 이해하기 위한 핵심 수학 기초. **행렬 곱연산**(LLM의 핵심 연산), **부동소수점**(FP32/BF16/FP8의 차이), 선형대수 기본 개념.

---

## 1. 행렬 곱연산 (Matrix Multiplication) — LLM의 엔진

LLM 추론의 95% 이상이 행렬 곱연산. GPU의 Tensor Core가 이것을 하드웨어로 가속.

### 기본 연산

```
A (m×k)  ×  B (k×n)  =  C (m×n)

A = [[1, 2],      B = [[5, 6],      C = [[1×5+2×7, 1×6+2×8],
     [3, 4]]           [7, 8]]            [3×5+4×7, 3×6+4×8]]

                                      = [[19, 22],
                                         [43, 50]]

시간 복잡도: O(m × k × n) — 내적 연산 m×n번, 각각 k번 곱셈
```

### 내적 (Dot Product)

```
a = [1, 2, 3]
b = [4, 5, 6]

a · b = 1×4 + 2×5 + 3×6 = 4 + 10 + 18 = 32

= |a| × |b| × cos(θ)
```

**임베딩 유사도의 본질**: 두 벡터의 내적 → 방향이 같을수록 큰 값.

### LLM에서의 행렬 곱

```
Attention: Q·K^T → (seq_len × seq_len) 유사도 행렬
  Q: (seq_len × d_k)
  K: (seq_len × d_k)
  K^T: (d_k × seq_len)
  Q·K^T: (seq_len × seq_len) ← n² 행렬 → 긴 시퀀스 메모리 병목

FFN: x·W₁ → (batch × seq_len × d_ff)
  x:  (batch × seq_len × d_model)   e.g. (1 × 512 × 4096)
  W₁: (d_model × d_ff)              e.g. (4096 × 16384)
  결과: (1 × 512 × 16384)           ← 이 곱이 반복됨
```

### 배치 행렬 곱 (Batch MatMul)

GPU는 여러 행렬을 **동시에 병렬 계산**.

```python
import torch

# 배치 크기=32, 시퀀스=512, 차원=4096
X = torch.randn(32, 512, 4096)   # (batch, seq, d_model)
W = torch.randn(4096, 16384)     # (d_model, d_ff)

# 모든 32개 배치를 병렬로 한 번에 계산
out = X @ W   # (32, 512, 16384)
```

### GPU 행렬 연산 최적화 — 타일링(Tiling)

GPU 메모리 계층: **레지스터 → SRAM(Shared Memory) → HBM(고대역폭 메모리)**  
HBM 접근 비용이 SRAM의 수십 배 → **타일 단위로 SRAM에 올려 반복 연산**해 HBM I/O 최소화.

```
[단순 행렬 곱: A(M×K) × B(K×N)]
  A의 한 행 전체(K개), B의 한 열 전체(K개)를 HBM에서 읽어 내적 계산
  → M×N번 HBM 접근 (매우 느림)

[타일링]
  A를 (tile_M × K) 조각, B를 (K × tile_N) 조각으로 나눔
  조각을 SRAM에 올린 후 tile_M × tile_N 결과를 SRAM에서 계산
  → HBM 접근 횟수 tile 크기에 반비례하여 감소

실제 cuBLAS: 타일 크기를 GPU 아키텍처(Warp/Tensor Core)에 맞게 최적화
Flash Attention도 이 원리로 Attention 행렬의 HBM I/O를 O(n²) → O(n)으로 감소
```

### FLOPS — AI 성능 단위

```
FLOP (Floating Point Operation) = 부동소수점 연산 1회
TFLOPS = 초당 10¹² FLOP

행렬 곱 A(m×k) × B(k×n):
  FLOPs = 2 × m × k × n  (곱셈 + 덧셈)

예시: 70B 파라미터 모델 토큰 1개 생성
  ≈ 2 × 70B = 140 GFLOP
  H100 (989 TFLOPS FP16) → 이론상 ~7000 토큰/초
  (실제: 메모리 대역폭, 오버헤드로 훨씬 낮음)
```

---

## 2. 부동소수점 (Floating Point)

컴퓨터가 소수를 표현하는 방식. AI 연산의 정밀도·속도·메모리를 결정.

### IEEE 754 부동소수점 구조

```
FP32 (Single Precision, 32비트):
  [부호 1비트][지수 8비트][가수 23비트]
  표현 범위: ±1.18×10⁻³⁸ ~ ±3.4×10³⁸
  정밀도: ~7자리 십진수

FP16 (Half Precision, 16비트):
  [부호 1비트][지수 5비트][가수 10비트]
  표현 범위: ±6×10⁻⁵ ~ ±65504   ← 범위가 좁음! 오버플로 주의

BF16 (Brain Float 16, Google):
  [부호 1비트][지수 8비트][가수 7비트]
  표현 범위: FP32와 같음 (지수 비트 동일)
  정밀도: 낮음, 하지만 범위 안전
```

```
수 π = 3.14159265358979...

FP32:  3.1415927   ← 7자리
FP16:  3.140625    ← 4자리, 범위 제한
BF16:  3.140625    ← 3자리이지만 범위 안전
```

### AI에서 쓰이는 부동소수점 비교

| 타입 | 비트 | 지수 | 가수 | 표현 범위 | 정밀도 | 용도 |
|---|---|---|---|---|---|---|
| **FP64** | 64 | 11 | 52 | 매우 큼 | 매우 높음 | 과학 계산 (AI에선 거의 안 씀) |
| **FP32** | 32 | 8 | 23 | 큰 범위 | 7자리 | 학습 기본, 참조 정밀도 |
| **TF32** | 19 | 8 | 10 | FP32 범위 | FP16 정밀도 | NVIDIA A100+ 학습 가속 |
| **BF16** | 16 | 8 | 7 | FP32 범위 | 낮음 | **LLM 학습/추론 표준** ⭐ |
| **FP16** | 16 | 5 | 10 | 좁음 | 보통 | 구형 추론, 오버플로 주의 |
| **FP8 E4M3** | 8 | 4 | 3 | 중간 | 낮음 | Blackwell 추론 (정밀도 중시) |
| **FP8 E5M2** | 8 | 5 | 2 | 큰 범위 | 매우 낮음 | Blackwell 학습 (그래디언트) |
| **INT8** | 8 | — | — | -128~127 | 정수 | 양자화 추론 |
| **INT4** | 4 | — | — | -8~7 | 정수 | 극한 양자화 (GPTQ/AWQ) |
| **NF4** | 4 | — | — | 특수 | — | QLoRA (정규분포 최적화) |

### 왜 BF16이 LLM 표준인가

```
FP16의 문제:
  지수 비트 5개 → 표현 가능한 최대값 65504
  학습 중 그래디언트가 이를 초과 → Overflow (NaN 발생)
  Loss Scaling 기법 필요 → 복잡

BF16의 해결:
  지수 비트 8개 (FP32와 동일) → 범위는 FP32와 같음
  가수 비트가 적어 정밀도는 낮지만,
  딥러닝에서 7자리 정밀도가 3자리보다 실제 품질 차이 거의 없음
  → Loss Scaling 불필요, 구현 단순, 메모리 절반
```

### 혼합 정밀도 (Mixed Precision)

```python
# PyTorch AMP (Automatic Mixed Precision)
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()  # FP16 사용 시 Loss Scaling

with autocast(dtype=torch.bfloat16):  # 순전파: BF16
    output = model(input)
    loss = criterion(output, target)

# 역전파: FP32로 그래디언트 누적 (파라미터 업데이트 정밀도 유지)
scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
```

```
순전파 (Forward):   BF16 → GPU 연산 빠름, 메모리 절반
가중치 마스터 복사: FP32 → 정밀한 파라미터 업데이트 (optimizer state)
역전파 (Backward):  BF16 or FP32
```

### 수치 안정성 문제

```python
# 작은 수의 덧셈 손실 (Catastrophic Cancellation)
a = 1000000.0  # FP32
b = 0.000001   # FP32

a + b = 1000000.0  # b가 사라짐 (정밀도 손실)

# Attention에서 softmax 수치 안정화
# Naive: exp(x) → 큰 x에서 overflow
# 안정화: exp(x - max(x)) → 범위 제한 후 softmax
```

---

## 3. 선형대수 핵심 개념

### 벡터

```python
# n차원 공간의 점 / 방향
v = [1.0, 2.0, 3.0]   # 3차원 벡터
                        # LLM 임베딩 = 4096차원 벡터

# 크기 (L2 Norm)
|v| = √(1² + 2² + 3²) = √14 ≈ 3.74

# 단위 벡터 (크기=1로 정규화)
v_hat = v / |v| = [0.267, 0.535, 0.802]
```

### 행렬 변환

```
행렬 W는 벡터를 다른 공간으로 변환하는 함수:

y = W · x
  = (다른 공간의 벡터)

LLM에서:
  Attention Q/K/V 변환: x → Q = x·W_Q  (입력 → 쿼리 공간)
  FFN: x → x·W₁ → ReLU → x·W₂  (특징 변환)
  Output: hidden → logits = hidden·W_vocab  (숨겨진 상태 → 어휘 확률)
```

### 전치 (Transpose)

```
A = [[1, 2, 3],      A^T = [[1, 4],
     [4, 5, 6]]              [2, 5],
                              [3, 6]]
(m×n) → (n×m)

Attention에서: Q·K^T → K를 전치해 내적 계산
```

### 행렬식 (Determinant) & 역행렬

```
det(A) ≠ 0 → 역행렬 존재 → 변환이 가역적
det(A) = 0 → 역행렬 없음 → 정보 손실 (차원 축소)

AI에서는 직접 사용보다 개념 이해용:
  LayerNorm이 activation 분포를 정규화 = 스케일 복원
```

---

## 4. 미분 / 그래디언트 — 학습의 원리

### 경사 하강법 (Gradient Descent)

```
손실 함수 L(θ)를 최소화하는 파라미터 θ 찾기

업데이트 규칙:
  θ ← θ - η × ∇L(θ)

η = 학습률 (learning rate)
∇L(θ) = 손실의 그래디언트 (기울기)
        = "θ를 조금 바꾸면 L이 얼마나 변하는가"
```

```
직관:
  산의 현재 위치 = 파라미터
  고도 = 손실
  경사 방향 = 그래디언트
  경사 반대로 한 발씩 내려감 = 경사 하강
```

### 역전파 (Backpropagation)

```
순전파: 입력 → 레이어 → 출력 → 손실 계산
역전파: 손실 → 체인 룰로 각 레이어의 그래디언트 계산

체인 룰: ∂L/∂x = ∂L/∂y × ∂y/∂x
```

### 옵티마이저

| 옵티마이저 | 특징 | LLM 학습 사용 |
|---|---|---|
| SGD | 단순, 느린 수렴 | 거의 안 씀 |
| Adam | 적응형 lr, 빠른 수렴 | 일반 DL |
| **AdamW** | Adam + Weight Decay 분리 | **LLM 표준** ⭐ |
| Adafactor | 메모리 효율 (optimizer state 축소) | 초대형 모델 |

---

## 5. 관련
- [[Transformer-Architecture]] · [[ML-Fundamentals]] · [[Inference-Optimization]]
