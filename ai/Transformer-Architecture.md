---
tags:
  - ai
  - transformer
  - attention
  - architecture
created: 2026-06-16
---

# Transformer Architecture

> [!summary] 한 줄 요약
> 2017년 "Attention Is All You Need" 논문에서 제안된 아키텍처 [Vaswani et al., 2017]. **Self-Attention 메커니즘**으로 입력 시퀀스 내 모든 위치 간 관계를 병렬로 계산해, RNN의 순차 처리 한계를 극복. GPT·Claude·Llama 등 현대 LLM의 기반.

---

## 1. 전체 구조

```
         Encoder (BERT류)            Decoder (GPT류)
         ┌─────────────┐             ┌─────────────────┐
입력     │  Embedding   │   출력토큰  │   Embedding      │
토큰 ──► │  + Positional│         ──►│  + Positional    │
         │  Encoding    │             │  Encoding        │
         └──────┬───────┘             └────────┬─────────┘
                │                              │
         ┌──────▼───────┐ ×N         ┌────────▼─────────┐ ×N
         │ Multi-Head   │            │ Masked Multi-Head │
         │ Self-Attention│            │ Self-Attention    │
         ├──────────────┤            ├──────────────────┤
         │ Feed Forward │            │ Feed Forward      │
         └──────────────┘            └──────────────────┘
                │                              │
         ┌──────▼───────┐             ┌────────▼─────────┐
         │ [CLS] vector │             │ Linear + Softmax  │
         │  (분류용)     │             │ 다음 토큰 확률    │
         └──────────────┘             └──────────────────┘
```

- **Encoder-only** (BERT, RoBERTa): 문장 전체를 보고 표현 추출 → 분류, 임베딩
- **Decoder-only** (GPT, Claude, Llama): 이전 토큰만 보고 다음 토큰 예측 → 생성 LLM
- **Encoder-Decoder** (T5, BART): 인코더 입력 + 디코더 생성 → 번역, 요약

---

## 2. Tokenization — 텍스트 → 숫자

```
입력: "안녕하세요"
       ↓ Tokenizer (SentencePiece / BPE)
토큰:  ["안녕", "하세", "요"]
       ↓ vocab lookup
ID:    [15234, 8821, 332]
       ↓ Embedding Layer
벡터:  [[0.12, -0.34, ...], [0.88, 0.02, ...], ...]  # (3, d_model)
```

- **BPE (Byte Pair Encoding)**: 자주 등장하는 바이트 쌍을 반복 병합 → 서브워드 단위
- **한국어**: 영어보다 토큰 효율이 낮음 (같은 내용이 더 많은 토큰 소모)
- **vocab size**: 보통 32K~128K 토큰

---

## 3. Positional Encoding — 순서 정보 주입

Self-Attention은 순서를 모른다 → 위치 정보를 임베딩에 더해 주입 [Vaswani et al., 2017].

```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

| 방식 | 예시 | 특징 |
|---|---|---|
| **Sinusoidal (고정)** | 원래 Transformer | 학습 없음, 외삽 불가 |
| **Learned** | BERT, GPT-2 | 학습, 고정 길이 |
| **RoPE** (Rotary PE) | LLaMA, Claude, Mistral | 상대 거리 기반, 외삽 기반 확장에 적합 ⭐ [Su et al., 2022] |
| **ALiBi** | MPT, Falcon | 어텐션 score에 선형 bias 추가, 외삽 우수 |

현대 LLM은 대부분 **RoPE**. 단, 기본 RoPE 그대로는 학습 시 본 길이를 크게 벗어나면 성능 저하 → 장문 컨텍스트는 아래 확장 기법 필요.

### RoPE 장문 컨텍스트 확장 기법

```
[기본 RoPE 한계]
  학습 컨텍스트 4096 토큰 → 추론 시 8192 토큰 → 위치 임베딩 out-of-distribution → 성능 급락

[해결: 위치 보간 (Position Interpolation)]
  학습 범위 내로 위치 ID를 압축 (스케일링)
  8192 토큰 → 4096 범위로 선형 압축 → 분포 유지

[YaRN (Yet another RoPE extensioN)]
  주파수별로 다르게 스케일링 (고주파는 더 보존, 저주파는 더 압축)
  + NTK-aware scaling 결합
  → Llama 3.1 128K, Mistral 128K에서 사용

[LongRoPE]
  비균일 위치 보간 + 단계적 파인튜닝
  → Phi-3-mini 128K, Phi-3.5-MoE 128K

[Dynamic NTK]
  추론 시 실제 시퀀스 길이에 맞게 동적으로 theta 조정
  → 추가 학습 없이도 어느 정도 외삽 가능
```

| 기법 | 추가 학습 필요 | 효과 | 사용 모델 |
|---|---|---|---|
| Linear Interpolation | △ 권장 | 기본 | 초기 연구 |
| NTK-aware Scaling | △ | 중간 | llama.cpp 구현 |
| **YaRN** | ✅ | 좋음 | Llama 3.1, Mistral |
| **LongRoPE** | ✅ | 좋음 | Phi-3.x |
| Dynamic NTK | ❌ | 보통 | 추론 엔진 |

---

## 4. Self-Attention — 핵심 메커니즘

각 토큰이 **시퀀스 내 다른 모든 토큰을 얼마나 참조할지** 가중치를 학습 [Vaswani et al., 2017].

```
입력 X (seq_len, d_model)
    ↓ 3개의 선형 변환
Q = X · W_Q   (Query  — 내가 무엇을 찾는가)
K = X · W_K   (Key    — 나는 어떤 정보를 가졌는가)
V = X · W_V   (Value  — 참조될 때 어떤 값을 줄 것인가)

Attention(Q, K, V) = softmax( Q·K^T / √d_k ) · V
```

```
Q·K^T 행렬 (seq_len × seq_len) — "토큰 간 유사도 행렬"
/√d_k — 스케일링 (그래디언트 소실 방지)
softmax — 확률 분포화
× V — 가중 합계 → 각 위치의 새 표현
```

**시간 복잡도**: O(n²·d) — 시퀀스 길이 n이 늘면 메모리·연산이 제곱으로 증가 → 긴 컨텍스트의 병목

---

## 5. Multi-Head Attention (MHA)

Self-Attention을 h개의 **헤드**로 병렬 수행 → 다양한 관계 동시 학습.

```
MultiHead(Q,K,V) = Concat(head_1, ..., head_h) · W_O

head_i = Attention(Q·W_Q_i, K·W_K_i, V·W_V_i)
```

- head 하나는 문법적 의존성, 다른 head는 의미적 연관, 또 다른 head는 공동 참조 등을 학습
- 현대 LLM: d_model=4096~8192, head 수=32~64

### GQA / MQA — KV Cache 최적화

| 방식 | 설명 | KV Cache |
|---|---|---|
| MHA (Multi-Head Attention) | Q, K, V 모두 head 수만큼 | head 수 × 2 |
| **GQA** (Grouped Query Attention) | K, V는 그룹별로 공유 | ÷ 그룹 수 |
| MQA (Multi-Query Attention) | K, V를 1개만 | 최소 |

LLaMA 3, Mistral, Claude 3+ → GQA 채택 [Ainslie et al., 2023]. [[Inference-Optimization]] KV Cache 절감. GQA는 기존 MHA 체크포인트로부터 5% 사전학습 연산만으로 업트레이닝 가능하다.

---

## 6. Feed-Forward Network (FFN)

Attention 이후 각 위치에 독립적으로 적용되는 2층 MLP.

```
FFN(x) = max(0, x·W_1 + b_1) · W_2 + b_2
        (또는 SiLU / GELU 활성화)

d_ff = 4 × d_model (보통)
```

- Attention이 **위치 간 정보 교환**이라면, FFN은 **각 위치의 특징 변환**
- 모델 파라미터의 약 2/3가 FFN에 집중 (d_model=4096 → d_ff=16384)

### SwiGLU (현대 LLM 표준)

```
SwiGLU(x, W, V) = SiLU(x·W) ⊙ (x·V)
```

LLaMA, PaLM, Claude 등 최신 모델의 FFN 활성화 함수 [Shazeer, 2020]. Vanilla ReLU 대비 성능 우수하며, SwiGLU를 사용할 경우 FFN 은닉 차원을 표준 4× 대신 8/3×d_model로 조정하는 것이 관례다.

---

## 7. Layer Normalization

Attention과 FFN 전/후에 적용. 학습 안정화.

```
LayerNorm(x) = (x - μ) / (σ + ε) * γ + β
```

- **Pre-LN** (최신 표준): Sub-layer 입력에 먼저 적용 → 학습 안정, 깊은 모델 가능
- **Post-LN** (원래 Transformer): Sub-layer 출력에 적용 → 학습 불안정 가능
- **RMSNorm** (LLaMA): 평균 제거 없이 RMS만 사용 → 연산 경량화 [Zhang & Sennrich, 2019]

---

## 8. Causal Masking (Decoder)

Decoder-only LLM은 **미래 토큰을 볼 수 없어야** 한다.

```
Attention mask:
     t1   t2   t3   t4
t1 [  1    0    0    0  ]
t2 [  1    1    0    0  ]
t3 [  1    1    1    0  ]
t4 [  1    1    1    1  ]
```

- 0인 위치는 softmax 전에 -∞로 마스킹 → softmax 후 0
- 학습 시 모든 위치를 병렬로 계산 (Teacher Forcing)
- 추론 시 자기회귀(autoregressive): 토큰을 하나씩 생성

---

## 9. Scaling Laws

| 요소 | 효과 |
|---|---|
| 파라미터 수 (N) | 증가할수록 성능↑ (log-linear) |
| 학습 토큰 수 (D) | 증가할수록 성능↑ |
| 연산량 (C = 6ND) | N·D 균형이 중요 |

**Chinchilla Optimal** [Hoffmann et al., 2022]:  
→ 최적 D/N ≈ 20 (파라미터 1개당 20토큰 학습)  
→ 70B 모델 → 약 1.4T 토큰 학습이 최적. Chinchilla(70B)는 Gopher(280B)·GPT-3(175B) 대비 동일 연산 예산에서 더 높은 성능을 달성하며 이 법칙을 입증했다.

**Emergent abilities**: 모델이 충분히 커지면 in-context learning, chain-of-thought 등 능력이 **갑자기 발현**.

---

## 10. MoE (Mixture of Experts)

Dense 모델의 FFN 대신 **여러 Expert FFN**을 두고, 라우터가 top-k를 선택 실행.

```
MoE Layer:
  입력 → Router (Softmax) → 상위 2개 Expert 선택
                          ↓
                  Expert_3, Expert_7 실행 → 가중 합산
```

- **활성 파라미터 < 전체 파라미터**: Mixtral 8×7B → 전체 47B이지만 추론 시 13B만 활성
- 같은 연산으로 더 많은 파라미터 효과 → 비용 대비 성능 우수
- **단점**: 라우팅 불균형(load balancing), 멀티노드 분산 복잡성

---

## 11. 관련
- [[LLM]] · [[Fine-Tuning]] · [[Inference-Optimization]] · [[Embedding]]

---

## 참고문헌

[1] A. Vaswani et al., "Attention Is All You Need," *NeurIPS*, 2017. arXiv:1706.03762
[2] J. Su et al., "RoFormer: Enhanced Transformer with Rotary Position Embedding," *Neurocomputing*, 2024. arXiv:2104.09864
[3] J. Ainslie et al., "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints," *EMNLP*, 2023. arXiv:2305.13245
[4] J. Hoffmann et al., "Training Compute-Optimal Large Language Models," *NeurIPS*, 2022. arXiv:2203.15556
[5] N. Shazeer, "GLU Variants Improve Transformer," 2020. arXiv:2002.05202
[6] B. Zhang & R. Sennrich, "Root Mean Square Layer Normalization," *NeurIPS*, 2019. arXiv:1910.07467
[7] R. Child et al., "Generating Long Sequences with Sparse Transformers," 2019. arXiv:1904.10509
