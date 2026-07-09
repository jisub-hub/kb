---
tags:
  - ai
  - fine-tuning
  - lora
  - rlhf
  - dpo
created: 2026-06-16
---

# Fine-Tuning

> [!summary] 한 줄 요약
> 사전학습된 LLM을 **특정 도메인·태스크·스타일**에 맞게 추가 학습하는 기법. 풀 파인튜닝부터 파라미터 효율적 기법(LoRA/QLoRA), 정렬 기법(RLHF/DPO)까지 목적에 따라 선택이 달라진다.

---

## 1. 언제 파인튜닝이 필요한가

```
프롬프트 엔지니어링 → RAG → 파인튜닝 → 처음부터 학습
      쉬움/빠름 ←─────────────────────────→ 비쌈/느림
```

| 상황 | 권장 방법 |
|---|---|
| 일반 도메인 지식 추가 | [[RAG]] (파인튜닝보다 효율적) |
| 특정 출력 **형식/스타일** 고정 | 파인튜닝 (SFT) |
| 도메인 특화 언어/용어 이해 | 파인튜닝 (SFT) |
| 유해 응답 제거, 안전성 | RLHF / DPO |
| 추론 능력 향상 | SFT + DPO (think step-by-step 데이터) |

> [!warning] 파인튜닝은 만능이 아니다
> "지식을 주입하려고 파인튜닝" → 대부분 RAG가 더 낫다.
> 파인튜닝은 **행동 패턴·형식·스타일** 변경에 효과적이다.

---

## 2. SFT (Supervised Fine-Tuning) — 지도 학습

**가장 기본적인 파인튜닝.** 사람이 작성한 (입력, 정답 출력) 쌍으로 학습.

```
데이터: [(질문, 좋은 답변), (질문, 좋은 답변), ...]
손실:   Cross-Entropy Loss (다음 토큰 예측)
```

### 데이터 형식 (Chat Template)

```jsonl
{"messages": [
  {"role": "system", "content": "당신은 법률 전문 AI입니다."},
  {"role": "user", "content": "계약서 검토를 부탁합니다."},
  {"role": "assistant", "content": "네, 검토해드리겠습니다. 계약서를 업로드해주세요."}
]}
```

### 데이터 품질 원칙
- 양보다 **질** — 저품질 1만개 < 고품질 1천개
- **다양성** — 동일 패턴의 반복은 과적합 유발
- **일관성** — 응답 스타일/형식 통일
- **최소 수량** 가이드: 형식 변경 ~100개, 도메인 적응 1K~10K, 역량 추가 10K+

---

## 3. LoRA (Low-Rank Adaptation) — 파라미터 효율적 학습

전체 가중치를 학습하는 대신 **저랭크 행렬 쌍만 추가 학습** [Hu et al., 2022].

```
원래 가중치 행렬 W (d × d) — 고정(frozen)
            ↓
  추가 학습: W' = W + ΔW = W + B·A

  B: (d × r), A: (r × d)   r << d (rank)
  파라미터 수: d² → 2·d·r (r=16이면 ~1/256)
```

초기화 시 A는 가우시안, B는 0으로 설정하여 학습 시작 시 ΔW=0을 보장한다.

```
예시: d=4096, r=16
  원래: 4096×4096 = 16.7M 파라미터
  LoRA: 4096×16 + 16×4096 = 131K 파라미터 (~0.8%)
```

| LoRA 하이퍼파라미터 | 의미 | 권장값 |
|---|---|---|
| `r` (rank) | 저랭크 차원. 클수록 표현력↑, 비용↑ | 8~64 |
| `lora_alpha` | 스케일링 계수 (α/r이 실제 lr) | r과 같거나 2배 |
| `target_modules` | 적용할 레이어 (q_proj, v_proj 등) | q, v 최소; q, k, v, o 권장 |
| `lora_dropout` | 과적합 방지 | 0.05~0.1 |

### LoRA 추론 — 가중치 병합

```python
# 추론 시 어댑터 on-the-fly 적용 또는 사전 병합
# 병합: W_merged = W + (alpha/r) · B·A
# → 추론 오버헤드 없음, 단 여러 어댑터 동적 전환 불가
```

---

## 4. QLoRA — 양자화 + LoRA

4-bit 양자화로 **기본 모델을 압축**하고, LoRA 어댑터만 BF16으로 학습 [Dettmers et al., 2023].

```
기본 모델: FP16 (140GB) → NF4 4-bit 양자화 (~35GB)
LoRA 어댑터: BF16 (별도, 작음)
→ 70B 모델을 단일 A100 80GB (또는 M5 Max 128GB)에서 파인튜닝 가능
```

```python
from transformers import BitsAndBytesConfig, AutoModelForCausalLM
from peft import LoraConfig, get_peft_model

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",          # Normal Float 4 (QLoRA 핵심)
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,     # 2중 양자화로 메모리 추가 절감
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70B-Instruct",
    quantization_config=bnb_config,
    device_map="auto"
)

lora_config = LoraConfig(
    r=16, lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.05, bias="none",
    task_type="CAUSAL_LM"
)
model = get_peft_model(model, lora_config)  # LoRA 레이어 삽입
```

---

## 5. RLHF (Reinforcement Learning from Human Feedback)

인간 피드백으로 **모델 행동을 정렬(alignment)**. GPT-4, Claude의 핵심 [Christiano et al., 2017].

```
3단계:
1. SFT   — 기본 지시 따르기 능력 부여
2. RM 학습 — 인간이 응답 쌍을 비교 선호 → Reward Model 학습
3. PPO   — RL로 Reward를 최대화하도록 SFT 모델 추가 학습
            (+ KL 발산 페널티로 SFT에서 너무 벗어나지 않게)
```

**단점**: PPO 학습이 불안정, 느림, 구현 복잡, Reward Hacking 위험.

---

## 6. DPO (Direct Preference Optimization)

RLHF에서 **Reward Model과 RL을 제거**하고, 선호 쌍 데이터로 직접 학습 [Rafailov et al., 2023]. DPO는 최적 정책과 보상 모델 사이의 수학적 관계를 활용해 RLHF 목적함수를 단순한 이진 분류 손실로 재표현한다.

```
데이터: (prompt, chosen 응답, rejected 응답)
손실: log σ(β·(log π_θ(chosen) - log π_θ(rejected) - 같은항_reference))
```

```python
from trl import DPOTrainer

# 데이터: {"prompt": ..., "chosen": ..., "rejected": ...}
trainer = DPOTrainer(
    model=model,
    ref_model=ref_model,   # frozen SFT 모델 (KL 기준)
    args=training_args,
    beta=0.1,              # KL 페널티 강도 (낮을수록 더 자유롭게 이탈)
    train_dataset=dataset,
    tokenizer=tokenizer,
)
trainer.train()
```

| | RLHF | DPO |
|---|---|---|
| 복잡성 | 높음 (RM + PPO) | 낮음 (단일 학습) |
| 안정성 | 불안정 | 안정적 |
| 성능 | 높음 | RLHF에 근접 |
| 데이터 | 비교 쌍 + 절대 점수 | 비교 쌍만 |

---

## 7. 선호 학습 최신 기법

| 기법 | 특징 | 참고 |
|---|---|---|
| **DPO** | 가장 단순, 안정적 | 표준 |
| **SimPO** | 참조 모델 불필요 | DPO 개선 |
| **ORPO** | SFT + 정렬 동시에 | 학습 1단계로 축소 [Hong et al., 2024] |
| **KTO** | rejected 없이 이진 피드백만 | 데이터 수집 쉬움 |
| **IPO** | 과최적화 방지 수식 개선 | DPO 변형 |

---

## 8. 전체 학습 파이프라인

```
1. 베이스 모델 선택
   └─ 사전학습된 base (지시 미학습) 또는 Instruct 모델 중 선택

2. SFT (Supervised Fine-Tuning)
   └─ 도메인 데이터 + 지시 따르기 데이터로 QLoRA 학습

3. 평가 (Evals)  ─── [[Evaluation]]
   └─ 태스크 성능 + 안전성 + 형식 준수 검증

4. 정렬 (선택)
   └─ DPO로 선호 응답 강화, 유해 응답 약화

5. 병합 & 양자화
   └─ LoRA 어댑터 fuse → GGUF/AWQ/GPTQ 양자화 → 배포
```

---

## 9. 주의사항 — Catastrophic Forgetting

파인튜닝의 핵심 리스크: **새 데이터를 배우면서 사전학습 능력을 잃어버리는 현상** [McCloskey & Cohen, 1989].

```
파인튜닝 전:  수학 80점 / 코딩 75점 / 법률 30점
도메인 SFT 후: 수학 55점 / 코딩 60점 / 법률 85점  ← 법률은 올랐지만 나머지 하락
```

### 원인
- SFT 손실이 새 도메인만 최적화 → 기존 능력의 가중치가 덮어씌워짐
- 학습률이 높거나 epoch가 많을수록 심해짐

### 완화 방법

| 방법 | 설명 |
|---|---|
| **낮은 학습률** | 1e-5 이하, 기존 가중치 변화 최소화 |
| **LoRA** | 원래 가중치 frozen → 망각 자체가 구조적으로 억제 ⭐ |
| **Replay / 혼합 데이터** | 학습 데이터에 기존 영역 데이터 20~30% 섞기 |
| **적은 epoch** | 1~3 epoch; 과도한 학습이 망각 심화 |
| **Early Stopping** | Validation loss로 적정 시점 탐지 |

> [!tip] LoRA가 망각에 강한 이유
> 원래 가중치 W는 완전히 고정(frozen). 새로 추가된 A·B 행렬만 학습.
> → 기존 능력은 W에 그대로, 새 능력은 ΔW=B·A에 분리 저장.

### 검증 방법

```python
# 파인튜닝 전후 범용 능력 비교 필수
benchmarks = ["MMLU", "HumanEval", "GSM8K"]  # 기존 능력 측정
domain_eval = ["law_qa", "contract_review"]    # 목표 능력 측정

# 도메인 성능↑ + 범용 성능 유지 → 성공
# 도메인 성능↑ + 범용 성능↓↓ → 망각 발생 → 학습률/epoch 조정
```

---

## 10. Apple M5 Max에서 파인튜닝 (MLX)

참고: [[Apple-M5-Max]] 섹션 6 — `mlx_lm.lora`로 70B LoRA 학습 가능.

```bash
python -m mlx_lm.lora \
  --model mlx-community/Meta-Llama-3.1-70B-Instruct-4bit \
  --train --data ./data \
  --iters 2000 --lora-layers 16
```

---

## 10. 관련
- [[LLM]] · [[Transformer-Architecture]] · [[Inference-Optimization]] · [[Evaluation]]
- [[Parametric-Knowledge-Editing]] — 전체 학습 대신 모듈/국소 가중치만 편집·핫스왑(LoRA 핫스왑 포함)
- [[Apple-M5-Max]] · [[DGX-Spark]]

---

## 참고문헌

[1] E. J. Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models," *ICLR*, 2022. arXiv:2106.09685
[2] T. Dettmers et al., "QLoRA: Efficient Finetuning of Quantized LLMs," *NeurIPS*, 2023. arXiv:2305.14314
[3] P. F. Christiano et al., "Deep Reinforcement Learning from Human Preferences," *NeurIPS*, 2017. arXiv:1706.03741
[4] R. Rafailov et al., "Direct Preference Optimization: Your Language Model is Secretly a Reward Model," *NeurIPS*, 2023. arXiv:2305.18290
[5] J. Hong et al., "ORPO: Monolithic Preference Optimization without Reference Model," *EMNLP*, 2024. arXiv:2403.07691
[6] M. McCloskey & N. J. Cohen, "Catastrophic Interference in Connectionist Networks: The Sequential Learning Problem," *Psychology of Learning and Motivation*, vol. 24, pp. 109–165, 1989.
