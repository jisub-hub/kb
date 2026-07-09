---
tags:
  - ai
  - inference
  - kv-cache
  - quantization
  - vllm
  - optimization
created: 2026-06-16
---

# Inference Optimization

> [!summary] 한 줄 요약
> LLM 추론을 **빠르게·저렴하게·대규모로** 처리하기 위한 기법 모음. KV Cache, 양자화, Speculative Decoding, Continuous Batching 등이 핵심. 프로덕션 추론 서버의 성능을 결정한다.

---

## 1. 추론의 두 단계 — Prefill vs Decode

```
[Prefill 단계]
  입력 토큰 전체를 병렬로 처리 → KV Cache 생성
  특징: GPU FLOPS 활용 높음 (compute-bound), 단 한 번

[Decode 단계]
  토큰을 하나씩 자기회귀 생성
  특징: 각 스텝에서 전체 KV Cache를 읽어야 함 (memory bandwidth-bound)
        → 메모리 대역폭이 병목
```

**DGX Spark(273 GB/s) vs M5 Max(~600 GB/s)**의 추론 속도 차이가 여기서 발생.  
→ [[Apple-M5-Max]] 고대역폭이 decode 단계에서 유리한 이유.

---

## 2. KV Cache

Decode 단계에서 이전 토큰의 Key·Value를 **재계산 없이 재사용**.

```
[KV Cache 없이]
  스텝 n: 토큰 1~n 전체의 K, V를 매번 재계산
  → O(n²) 연산

[KV Cache 있이]
  스텝 n: 토큰 n의 K, V만 계산, 1~(n-1)은 캐시에서 읽음
  → O(n) 연산
```

### KV Cache 크기

```
KV Cache = 2 (K+V) × num_layers × num_heads × head_dim × seq_len × bytes_per_element

예시: Llama 3.1 70B, BF16, seq_len=4096
= 2 × 80 × 8 × 128 × 4096 × 2 bytes
= ~20GB
```

**seq_len이 늘어날수록 KV Cache가 선형 증가 → VRAM의 주요 소비자.**

### GQA로 KV Cache 절감

MHA → GQA(그룹=8) 전환 시 KV Cache 1/8로 감소.  
→ [[Transformer-Architecture]] 섹션 5 참고.

---

## 3. 양자화 (Quantization)

가중치/활성화를 낮은 정밀도로 표현해 메모리와 연산을 줄임.

### 정밀도 비교

| 타입 | 비트 | 표현 범위 | 메모리 (70B 기준) | 특징 |
|---|---|---|---|---|
| FP32 | 32 | 큰 범위 | 280GB | 학습 기본값 |
| **BF16** | 16 | FP32와 같은 지수 | 140GB | 추론/학습 표준 |
| FP8 | 8 | — | 70GB | Blackwell GPU 하드웨어 지원 |
| **INT8** | 8 | 정수 | 70GB | 속도 우수, 품질 소폭 저하 |
| **INT4 / NF4** | 4 | — | 35GB | 크게 압축, 허용 가능한 품질 |
| INT2 | 2 | — | 17GB | 품질 손상 심함 |

### 양자화 방식

#### PTQ (Post-Training Quantization) — 학습 후 변환

```python
# GPTQ [Frantar et al., 2023]: 역헤시안 기반, 레이어별 오류 최소화
# pip install auto-gptq
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

quantize_config = BaseQuantizeConfig(bits=4, group_size=128, desc_act=False)
model = AutoGPTQForCausalLM.from_pretrained("meta-llama/Llama-3.1-70B", quantize_config)
model.quantize(calibration_dataset)  # 소량의 캘리브레이션 데이터 필요
```

```python
# AWQ [Lin et al., 2024] (Activation-aware Weight Quantization): 중요 가중치 보호
# MLSys 2024 Best Paper Award
# pip install autoawq
from awq import AutoAWQForCausalLM

model = AutoAWQForCausalLM.from_pretrained("meta-llama/Llama-3.1-70B")
model.quantize(tokenizer, quant_config={"zero_point": True, "q_group_size": 128, "w_bit": 4})
```

#### GGUF — llama.cpp / Ollama 표준 포맷

```
Q2_K, Q3_K_M, Q4_K_M, Q5_K_M, Q6_K, Q8_0, F16
```

| GGUF 타입 | 70B 크기 | 품질 |
|---|---|---|
| Q4_K_M | ~40GB | 실용적 (권장) ⭐ |
| Q5_K_M | ~49GB | 좋음 |
| Q6_K | ~58GB | 매우 좋음 |
| Q8_0 | ~74GB | FP16 대비 손실 거의 없음 |

---

## 4. Speculative Decoding

**작은 Draft 모델**이 여러 토큰을 미리 예측 → **큰 Target 모델**이 병렬로 검증 [Leviathan et al., 2023]. 수학적으로 Target 모델의 출력 분포를 그대로 보존하면서 2~3× 속도 향상이 가능하다.

```
[일반 Decode]
  Big Model: 토큰1 → 토큰2 → 토큰3 → ...  (순차)

[Speculative Decoding]
  Small Draft: 빠르게 [t1, t2, t3, t4, t5] 예측
  Big Target:  5개를 한 번에 병렬 검증
               → 맞으면 5개 수용, 틀리면 첫 오류까지만 수용 후 재시작
```

- **속도 향상**: 2~4x (Draft 모델 품질과 도메인 일치도에 따라 달라짐)
- **정확도 무손실**: Target 모델 분포를 수학적으로 보존
- 대화형 서비스처럼 **반복 패턴이 많은** 워크로드에 효과적

---

## 5. Continuous Batching

기존 Static Batching의 문제: 배치 내 긴 요청이 짧은 요청들을 대기시킴. Orca는 iteration 단위 스케줄링(Continuous Batching)을 도입해 FasterTransformer 대비 36.9× 처리량 향상을 달성했다 [Yu et al., 2022].

```
[Static Batching]
  배치 = [req1(10 tok), req2(100 tok), req3(5 tok)]
  → req1, req3이 끝나도 req2 완료까지 전부 대기

[Continuous Batching]
  완료된 슬롯에 즉시 새 요청 투입 (iteration-level batching)
  → GPU 활용률 극대화, 대기 시간 최소화
```

vLLM, TGI 등 프로덕션 추론 서버의 핵심 기능.

---

## 6. PagedAttention (vLLM)

KV Cache를 OS 가상 메모리처럼 **비연속 페이지**로 관리 [Kwon et al., 2023].

```
[기존 KV Cache]
  각 요청에 최대 seq_len만큼 연속 메모리 예약 → 단편화 심함

[PagedAttention]
  KV Cache를 고정 크기 블록(페이지)으로 분할
  → 요청별로 필요한 만큼만 할당, 동적 확장
  → 메모리 낭비 감소, 배치 크기 3~4배 증가 가능
```

---

## 7. Flash Attention

**메모리 I/O를 최소화**하는 Attention 구현. 수학적으로 동일한 결과, 3~10x 속도 향상 [Dao et al., 2022].

```
[일반 Attention]
  Q·K^T → (seq_len×seq_len 행렬을 HBM에 저장) → softmax → ×V
  → HBM 읽기/쓰기: O(n²)

[Flash Attention]
  타일(tile) 단위로 쪼개 SRAM에서 처리
  → HBM 읽기/쓰기: O(n)
  → 메모리 절약 + 속도 향상 + 긴 시퀀스 가능
```

Flash Attention 2/3: 더 나은 타일링, H100 Tensor Core 최적화.

---

## 8. vLLM — 프로덕션 추론 서버

```bash
pip install vllm

# OpenAI 호환 서버 시작
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.1-70B-Instruct \
  --quantization awq \           # AWQ 양자화 적용
  --max-model-len 32768 \
  --tensor-parallel-size 2 \     # GPU 2개 텐서 병렬
  --gpu-memory-utilization 0.9

# curl 테스트
curl http://localhost:8000/v1/chat/completions \
  -d '{"model": "meta-llama/Llama-3.1-70B-Instruct", "messages": [...]}'
```

| vLLM 기능 | 설명 |
|---|---|
| PagedAttention | KV Cache 효율화 |
| Continuous Batching | 높은 처리량 |
| Tensor Parallelism | 다중 GPU 분산 |
| Speculative Decoding | 지연 감소 |
| Prefix Caching | 공통 시스템 프롬프트 캐시 |
| LoRA 동적 로딩 | 다중 어댑터 런타임 전환 |

---

## 9. Tensor Parallelism & Pipeline Parallelism

큰 모델을 여러 GPU에 분산.

```
[Tensor Parallelism]
  각 레이어의 가중치 행렬을 가로로 분할 → GPU 간 AllReduce
  → 레이어 수준 병렬, 통신 오버헤드 있음
  → 단일 노드 내 4~8 GPU에 적합

[Pipeline Parallelism]
  레이어 그룹을 GPU에 순서대로 배치 (레이어 1-20 → GPU1, 21-40 → GPU2)
  → 노드 간 분산 가능, Bubble 오버헤드 있음

[실제: Tensor + Pipeline 혼합]
  8 GPU/노드 × 4 노드 = 32 GPU → TP=8, PP=4
```

---

## 10. 추론 최적화 결정 트리

```
단일 GPU / 로컬:
  → llama.cpp (GGUF Q4_K_M) 또는 Ollama
  → Apple M5 Max: mlx_lm.server
  → NVIDIA GPU: vLLM + AWQ/GPTQ

소규모 프로덕션 (GPU 1~4):
  → vLLM + AWQ + Continuous Batching
  → 저지연 필요 → Speculative Decoding 추가

대규모 프로덕션 (GPU 8+):
  → vLLM + Tensor Parallelism
  → 또는 TGI (HuggingFace Text Generation Inference)

클라우드 관리형:
  → AWS Bedrock / Azure AI / NVIDIA NIM
  → DGX Spark: NIM 컨테이너 즉시 배포
```

---

## 11. 관련
- [[LLM]] · [[Transformer-Architecture]] · [[Fine-Tuning]]
- [[DGX-Spark]] · [[Apple-M5-Max]]

---

## 참고문헌

[1] W. Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," *SOSP*, 2023. arXiv:2309.06180
[2] T. Dao et al., "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness," *NeurIPS*, 2022. arXiv:2205.14135
[3] Y. Leviathan, M. Kalman & Y. Matias, "Fast Inference from Transformers via Speculative Decoding," *ICML*, 2023. arXiv:2211.17192
[4] E. Frantar et al., "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers," *ICLR*, 2023. arXiv:2210.17323
[5] J. Lin et al., "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration," *MLSys*, 2024. arXiv:2306.00978
[6] G.-I. Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models," *OSDI*, 2022.
