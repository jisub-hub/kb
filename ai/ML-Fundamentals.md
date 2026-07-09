---
tags:
  - ai
  - ml
  - machine-learning
  - regression
  - classification
  - bayesian
  - dataset
created: 2026-06-16
---

# ML Fundamentals

> [!summary] 한 줄 요약
> 머신러닝의 기초 개념. 데이터 분할 전략, 과적합/과소적합, Regression·Classification 태스크 유형, Bayesian 확률론적 사고까지. LLM을 이해하고 활용하기 위해 필요한 기반 개념.

---

## 1. 데이터셋 분할 — Train / Validation / Test

### 왜 분할하는가

```
모델이 학습 데이터를 "외우면" (과적합):
  학습 데이터 정확도: 99%
  실제 새 데이터 정확도: 60%   ← 이게 진짜 성능

→ 본 적 없는 데이터로 성능을 측정해야 한다
```

### 3개 분할의 역할

```
전체 데이터
  ├── Train (70~80%)    — 모델이 학습하는 데이터
  ├── Validation (10~15%) — 하이퍼파라미터 튜닝에 사용 (학습 중 확인)
  └── Test (10~15%)    — 최종 성능 평가 (단 한 번만 사용) ⭐
```

| 분할 | 역할 | 주의 |
|---|---|---|
| **Train** | 파라미터 업데이트 | 학습 손실 최소화 |
| **Validation** | 에폭 선택, 조기종료, 하이퍼파라미터 탐색 | Train에 포함 X |
| **Test** | 최종 일반화 성능 보고 | **절대 학습/튜닝에 사용 금지** |

> [!warning] Test Set 오염
> Test를 보고 모델을 수정하는 순간, Test는 사실상 Validation이 된다.
> 최종 보고 전까지 Test는 잠가두는 것이 원칙.

### 분할 비율 선택 기준

| 데이터 크기 | 권장 비율 | 이유 |
|---|---|---|
| 소규모 (~1K) | K-Fold CV | 단일 분할은 분산 너무 큼 |
| 중규모 (~10K) | 70/15/15 | 균형 |
| 대규모 (~1M+) | 98/1/1 | 1%도 충분한 양 |
| 불균형 클래스 | Stratified Split 필수 | 희귀 클래스 비율 유지 |

### Stratified Split (층화 분할)

불균형 데이터에서 단순 무작위 분할은 희귀 클래스가 특정 분할에 쏠릴 위험.

```python
from sklearn.model_selection import StratifiedShuffleSplit

# 클래스 비율: [정상 95%, 사기 5%]
# → 무작위 분할 시 Test에 사기 케이스가 0개일 수 있음
# → 층화 분할로 각 분할에서 5%를 유지

sss = StratifiedShuffleSplit(n_splits=1, test_size=0.2, random_state=42)
for train_idx, test_idx in sss.split(X, y):
    X_train, X_test = X[train_idx], X[test_idx]
    y_train, y_test = y[train_idx], y[test_idx]
```

### 교차 검증 (K-Fold Cross Validation)

데이터가 적을 때 사용. Train/Validation을 K번 다르게 나눠 평균냄.

```
K=5 일 때:
  폴드1: [V][T][T][T][T]
  폴드2: [T][V][T][T][T]
  폴드3: [T][T][V][T][T]
  폴드4: [T][T][T][V][T]
  폴드5: [T][T][T][T][V]
  
  성능 = 5번의 Validation 성능 평균
  → 분할 운에 의한 분산 제거
```

### 불균형 데이터 처리

```python
# 클래스 불균형 (정상:사기 = 95:5)
from sklearn.utils import class_weight

# 방법 1: class_weight='balanced' — 소수 클래스에 더 높은 손실 가중치
weights = class_weight.compute_class_weight('balanced', classes=[0,1], y=y_train)
# [0.526, 9.5]  → 사기(1)에 18배 높은 가중치

# 방법 2: 오버샘플링 (SMOTE)
from imblearn.over_sampling import SMOTE
X_resampled, y_resampled = SMOTE().fit_resample(X_train, y_train)

# 방법 3: 언더샘플링 — 다수 클래스 줄이기 (데이터 손실 위험)
```

### LLM Eval에서의 적용

LLM 파인튜닝도 동일 원칙:
- Train: 학습 JSONL
- Validation: 에폭별 loss 모니터링, 조기 종료 결정
- Test (Eval Suite): [[Evaluation]]의 최종 성능 측정

---

## 2. 과적합과 과소적합 (Bias-Variance Tradeoff)

```
과소적합 (Underfitting)          과적합 (Overfitting)
──────────────────────────────────────────────────
모델이 너무 단순                 모델이 너무 복잡
학습 데이터도 못 맞춤            학습 데이터는 완벽히 맞추나
                                 새 데이터에 일반화 실패
Train loss 높음                  Train loss 낮음
Val loss 높음                    Val loss 높음

해결: 모델 복잡도 증가            해결: 정규화, 드롭아웃, 더 많은 데이터
      더 많은 학습                       조기 종료 (Early Stopping)
```

### 조기 종료 (Early Stopping)

```
에폭   Train Loss   Val Loss
  1      0.90         0.88
  5      0.50         0.51
 10      0.30         0.32       ← 최적점
 15      0.20         0.38       ← 과적합 시작
 20      0.15         0.45

→ Val Loss가 상승하기 시작하면 학습 중단 → 에폭 10 모델 저장
```

---

## 3. Regression (회귀) — 연속값 예측

**출력이 연속적인 숫자**인 태스크.

```
입력: [집 크기, 방 수, 위치]  →  출력: 가격 (42,000,000)
입력: [기온, 습도, 풍속]     →  출력: 강수량 (12.3mm)
```

### 선형 회귀

```
y = w₁x₁ + w₂x₂ + ... + b

손실: MSE (Mean Squared Error)
  L = (1/n) Σ(y_pred - y_true)²
```

### 평가 지표

| 지표 | 수식 | 의미 |
|---|---|---|
| **MSE** | mean((pred-true)²) | 제곱 오차 평균. 이상치에 민감 |
| **RMSE** | √MSE | MSE의 제곱근. 단위가 원래와 같음 |
| **MAE** | mean(\|pred-true\|) | 절대 오차 평균. 이상치에 덜 민감 |
| **R²** | 1 - SS_res/SS_tot | 설명 분산 비율. 1에 가까울수록 좋음 |

### LLM과의 연관

- 임베딩 벡터로 회귀: 텍스트 → 벡터 → 점수 예측 (감성 점수, 난이도 등)
- LLM 자체는 내부적으로 다음 토큰의 확률을 계산 (softmax 출력)

---

## 4. Classification (분류) — 범주 예측

**출력이 이산적인 클래스**인 태스크.

```
이진(Binary):  "스팸인가?" → Yes/No
다중(Multi):   "어떤 주제?" → [정치, 경제, 스포츠, 연예]
다중 레이블:   "태그는?" → [Python, ML, Tutorial] (복수 선택)
```

### 로지스틱 회귀 (이진 분류)

```
z = w·x + b
p = sigmoid(z) = 1 / (1 + e^(-z))   → 0~1 확률

손실: Binary Cross-Entropy
  L = -[y·log(p) + (1-y)·log(1-p)]
```

### 다중 분류

```
z = W·x + b  (클래스 수만큼 출력)
p = softmax(z) = e^(zᵢ) / Σe^(zⱼ)   → 각 클래스 확률, 합이 1

손실: Categorical Cross-Entropy
  L = -Σ yᵢ·log(pᵢ)
```

### 평가 지표

```
Confusion Matrix:
                  예측 Positive   예측 Negative
실제 Positive         TP               FN
실제 Negative         FP               TN
```

| 지표 | 수식 | 의미 | 높아야 할 때 |
|---|---|---|---|
| **Accuracy** | (TP+TN)/(전체) | 전체 정확률 | 클래스 균형일 때 |
| **Precision** | TP/(TP+FP) | 양성 예측의 정확률 | 거짓 양성 비용 클 때 (스팸 필터) |
| **Recall** | TP/(TP+FN) | 실제 양성 탐지율 | 거짓 음성 비용 클 때 (암 진단) |
| **F1** | 2·P·R/(P+R) | Precision·Recall 조화평균 | 불균형 데이터셋 |
| **AUC-ROC** | ROC 곡선 면적 | 임계값 독립적 성능 | 이진 분류 일반 |
| **MCC** | 상관계수 기반 | 불균형에 가장 강건 | 극불균형 (1:99) |

> [!warning] 불균형 데이터에서 Accuracy의 함정
> 사기 탐지: 정상 95%, 사기 5%  
> "전부 정상으로 예측"하는 모델 → Accuracy 95% ← 의미 없는 수치  
> → **F1 또는 MCC를 주 지표로** 사용해야 실제 성능 파악 가능

### LLM에서의 분류

```python
# LLM으로 분류 태스크 수행
prompt = """다음 텍스트의 감성을 분류하라.
분류: POSITIVE, NEGATIVE, NEUTRAL 중 하나만 답하라.
텍스트: {text}"""

# 또는 Structured Output으로 강제
class Sentiment(BaseModel):
    label: Literal["POSITIVE", "NEGATIVE", "NEUTRAL"]
    confidence: float
```

---

## 5. Bayesian 확률론적 사고

### 베이즈 정리

```
P(A|B) = P(B|A) × P(A) / P(B)

사후 확률 = 우도 × 사전 확률 / 증거
(Posterior = Likelihood × Prior / Evidence)
```

**예시**: 스팸 필터
```
P(스팸 | "무료" 포함)
  = P("무료" 포함 | 스팸) × P(스팸) / P("무료" 포함)
  = 0.8 × 0.2 / 0.1
  = 1.6 → 정규화 후 약 0.94

→ "무료" 단어가 포함되면 스팸일 확률 94%
```

### LLM과 Bayesian 사고

LLM의 **토큰 생성은 본질적으로 조건부 확률**:

```
P(다음 토큰 | 이전 모든 토큰) = softmax(logits)

"오늘 날씨가 정말" → 다음 토큰 확률:
  "좋다": 0.35
  "나쁘다": 0.25
  "화창하다": 0.20
  기타: 0.20
```

**Temperature와 확률**:
```python
# temperature가 낮을수록 확률 분포가 날카로워짐
# temperature=0.1 → 최고 확률 토큰 거의 확정
# temperature=1.0 → 원래 분포 유지
# temperature=2.0 → 분포가 평평해짐 (창의적/무작위)

logits_adjusted = logits / temperature
probs = softmax(logits_adjusted)
```

### Naive Bayes 분류기

특징들이 독립이라고 가정하고 베이즈 정리 적용:

```
P(클래스 | 특징들) ∝ P(클래스) × Π P(특징ᵢ | 클래스)

장점: 빠름, 데이터 적어도 동작, 텍스트 분류에 효과적
단점: 특징 독립 가정이 현실에서 틀림
```

### 불확실성 표현

Bayesian 관점에서 LLM 출력을 다룰 때:

```python
# 모델이 확신이 없으면 낮은 확률 / 높은 entropy
# 높은 확신 → 낮은 entropy
import scipy.stats

probs = [0.9, 0.05, 0.03, 0.02]  # 확신 높음
entropy_high_conf = scipy.stats.entropy(probs)  # 낮음

probs = [0.26, 0.25, 0.25, 0.24]  # 확신 낮음
entropy_low_conf = scipy.stats.entropy(probs)   # 높음 → 환각 위험
```

---

## 6. 지도/비지도/강화학습

| 학습 유형 | 데이터 | 목표 | LLM 적용 |
|---|---|---|---|
| **지도학습** | (입력, 정답) 쌍 | 정답 맞추기 | SFT, Embedding 학습 |
| **비지도학습** | 정답 없는 데이터 | 패턴/구조 발견 | 사전학습(다음 토큰 예측) |
| **강화학습** | 행동·보상 | 누적 보상 최대화 | RLHF PPO 단계 |
| **자기지도학습** | 데이터 자체가 레이블 | 마스킹된 부분 복원 | BERT MLM, GPT CLM |

---

## 7. 관련
- [[LLM]] · [[Transformer-Architecture]] · [[Fine-Tuning]] · [[Evaluation]] · [[Math-Foundations]]
