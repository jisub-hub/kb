---
tags:
  - ai
  - evaluation
  - evals
  - ragas
  - benchmark
created: 2026-06-16
---

# Evaluation (LLM Evals)

> [!summary] 한 줄 요약
> LLM 애플리케이션의 **품질을 측정**하는 체계. "모델이 잘 작동하는가?"를 정량적으로 검증. 배포 전 회귀 방지, 모델 비교, 파인튜닝 효과 측정에 필수. 평가 없이는 개선도 없다.

---

## 1. 왜 Eval이 중요한가

```
Eval 없는 LLM 개발:
  프롬프트 수정 → 수동 테스트 몇 개 → "괜찮은 것 같은데" → 배포 → 프로덕션 장애

Eval 있는 LLM 개발:
  프롬프트 수정 → Eval Suite 자동 실행 → 수치 비교 → 통과 시 배포 → 안정
```

**전통 소프트웨어 테스트 vs LLM Eval**:
- 소프트웨어: 입력 → 정확히 일치하는 출력 검증
- LLM: 입력 → 의미적으로 올바른 출력 검증 (정확 매칭 불가)

---

## 2. Eval 유형

### 2.1. Code-Based (규칙 기반) — 빠르고 확실

```python
# 정규식, 포맷 검사, 키워드 포함 여부
def eval_json_format(output: str) -> bool:
    try:
        json.loads(output)
        return True
    except:
        return False

def eval_contains_citation(output: str) -> bool:
    return bool(re.search(r'\[\d+\]', output))   # [1], [2] 형태 인용 있는지
```

### 2.2. Human Eval — 가장 정확, 느리고 비쌈

- 사람이 직접 응답 채점 (1~5점 척도, 선호 쌍 비교)
- 최초 품질 기준선 수립, 모델 비교에 사용
- **병목**: 속도, 비용, 일관성 (평가자 간 불일치)

### 2.3. LLM-as-Judge — 실용적 균형 ⭐

**강력한 LLM(judge)이 다른 LLM의 응답을 채점** [Zheng et al., 2023]. MT-Bench와 Chatbot Arena를 통해 GPT-4 judge가 인간 평가자와 높은 일치도(>80%)를 보임이 검증됐다.

```python
JUDGE_PROMPT = """
다음 기준으로 응답을 1~5점으로 평가하라:
- 사실 정확성 (근거 있는 내용인가)
- 완전성 (질문에 완전히 답했는가)
- 간결성 (불필요한 내용 없는가)

질문: {question}
응답: {response}
참고 컨텍스트: {context}

점수(1-5)와 한 줄 이유를 JSON으로 답하라.
"""

def llm_judge(question, response, context):
    result = llm.call(JUDGE_PROMPT.format(...))
    return json.loads(result)  # {"score": 4, "reason": "..."}
```

**주의**: Judge LLM도 편향이 있다 (긴 응답 선호, 자기 자신 선호 등). 여러 Judge 앙상블 권장.

---

## 3. RAG Eval — RAGAS

RAG 파이프라인 전용 평가 프레임워크 [Es et al., 2023]. RAGAS는 ground-truth 인간 어노테이션 없이도 Faithfulness, Answer Relevancy, Context Recall, Context Precision 네 가지 차원을 LLM 기반으로 자동 측정한다.

```python
# pip install ragas
from ragas import evaluate
from ragas.metrics import (
    faithfulness,        # 충실성: 답변이 컨텍스트에 근거하는가
    answer_relevancy,    # 관련성: 답변이 질문에 답하는가
    context_recall,      # 재현율: 정답에 필요한 컨텍스트가 검색됐는가
    context_precision,   # 정밀도: 검색된 컨텍스트가 실제로 필요한가
)
from datasets import Dataset

data = {
    "question": ["Spring Boot 자동설정이란?"],
    "answer": ["...모델 답변..."],
    "contexts": [["...검색된 청크1...", "...청크2..."]],
    "ground_truth": ["...정답..."]
}
dataset = Dataset.from_dict(data)
result = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall, context_precision])
```

### RAGAS 4대 지표

| 지표 | 측정 | 범위 | 개선 방법 |
|---|---|---|---|
| **Faithfulness** (충실성) | 답변이 컨텍스트에서 지지되는가 | 0~1 | 프롬프트: "컨텍스트에 없으면 모름" |
| **Answer Relevancy** | 답변이 질문과 관련 있는가 | 0~1 | 프롬프트 명확화, 필터링 |
| **Context Recall** | 정답에 필요한 청크가 검색됐는가 | 0~1 | 청킹 전략, top-k 증가 |
| **Context Precision** | 검색 청크 중 실제 유용한 비율 | 0~1 | 리랭킹, 임베딩 모델 개선 |

---

## 4. 벤치마크 — 모델 능력 비교

| 벤치마크 | 측정 영역 | 비고 |
|---|---|---|
| **MMLU** [Hendrycks et al., 2021] | 57개 분야 지식 (대학원 수준) | 범용 지식 |
| **HumanEval / MBPP** [Chen et al., 2021] | 코드 생성 정확도 | 프로그래밍 |
| **GSM8K / MATH** | 수학 추론 | 단계적 사고 |
| **HellaSwag / Winogrande** | 상식 추론 | 언어 이해 |
| **TruthfulQA** | 사실성, 환각 저항 | 안전성 |
| **MT-Bench** [Zheng et al., 2023] | 다중 턴 대화 품질 | LLM-as-Judge 기반 |
| **LMSYS Chatbot Arena** | 실사용자 선호 투표 | 실전 체감 ⭐ |
| **KoBEST / KLUE** | 한국어 NLU | 한국어 특화 |

---

## 5. Eval 데이터셋 구축

```
좋은 Eval 데이터셋 조건:
1. 다양성 — 쉬운/어려운 케이스, 엣지 케이스 포함
2. 실제 사용자 질문 반영 — 프로덕션 로그에서 샘플링 ⭐
3. 명확한 정답 기준 — 모호한 케이스는 제외 또는 별도 처리
4. 정기 갱신 — 서비스 변화 반영
```

```python
# Eval 케이스 템플릿
eval_cases = [
    {
        "id": "case_001",
        "category": "factual",         # factual / reasoning / safety / format
        "difficulty": "hard",
        "input": "질문 또는 입력",
        "expected_output": "정답 또는 기준",
        "tags": ["korean", "legal"],
        "notes": "왜 이 케이스가 중요한가"
    }
]
```

---

## 6. CI/CD에 Eval 통합

```yaml
# .github/workflows/llm-eval.yml
jobs:
  eval:
    steps:
      - name: Run LLM Evals
        run: python run_evals.py --suite regression
      - name: Check thresholds
        run: |
          python check_metrics.py \
            --faithfulness-min 0.85 \
            --relevancy-min 0.80 \
            --fail-on-regression
```

```python
# run_evals.py
def run_regression_suite():
    results = []
    for case in load_eval_cases("regression_suite.json"):
        response = my_rag_pipeline(case["input"])
        score = evaluate_single(response, case)
        results.append(score)

    avg_score = mean(results)
    if avg_score < THRESHOLD:
        raise SystemExit(f"Eval failed: {avg_score:.2f} < {THRESHOLD}")
```

---

## 7. A/B 테스트 (모델/프롬프트 비교)

```python
# 두 버전의 응답을 LLM-Judge로 비교
def ab_test(question, response_a, response_b, judge_llm):
    # position bias 방지: A/B 순서를 절반씩 swap
    prompt = f"""
    두 응답 중 어느 것이 더 나은가? 이유와 함께 'A' 또는 'B'로만 답하라.
    질문: {question}
    A: {response_a}
    B: {response_b}
    """
    winner = judge_llm.call(prompt)
    return winner
```

### A/B 통계적 유의성

단순 평균 비교만으로는 충분하지 않음. 샘플 수가 적으면 우연일 수 있음.

```python
from scipy import stats

scores_a = [0.82, 0.79, 0.85, ...]  # 버전 A 케이스별 점수
scores_b = [0.88, 0.83, 0.91, ...]  # 버전 B 케이스별 점수

# 대응 t-검정 (같은 케이스로 두 버전 비교)
t_stat, p_value = stats.ttest_rel(scores_a, scores_b)

if p_value < 0.05:
    print(f"통계적으로 유의미한 차이 (p={p_value:.3f})")
else:
    print(f"차이가 우연일 수 있음 (p={p_value:.3f}), 샘플 더 필요")
```

> 실용 기준: **최소 50~100개 케이스** 이상에서 p < 0.05 이어야 신뢰할 수 있는 개선.

---

## 8. 프로덕션 드리프트 탐지 — "Eval 통과 vs 프로덕션 실패" 간극

7장까지는 **오프라인(골든셋) Eval**의 이야기였다. 그런데 현장에서 가장 흔한 사고는 이것이다:

```
오프라인 Eval Suite:  Faithfulness 0.89 ✅  Relevancy 0.86 ✅  → 배포
프로덕션 2주 후:      CS 문의 ↑, "답이 예전만 못하다", 재질문율 ↑
                     그런데 Eval을 다시 돌려도 점수는 그대로 높음 ⚠️
```

**점수는 통과하는데 실사용 품질이 떨어진다.** 이 간극의 원인을 진단하고 메우는 것이 프로덕션 Eval의 핵심이다.

### 8.1 온라인 드리프트 — 무엇이 변했나

오프라인 골든셋은 **고정**돼 있지만 프로덕션의 입력·구성 요소는 **계속 움직인다**. 점수가 그대로인데 체감 품질이 떨어진다면 거의 항상 아래 중 하나다.

| 드리프트 원인 | 무슨 일이 일어나는가 | 신호 |
|---|---|---|
| **모델 업데이트** | API 모델이 조용히 버전업 → 같은 프롬프트에 다른 출력 | 배포 안 했는데 출력 톤·형식 변화 |
| **프롬프트 변경** | 다른 팀원이 시스템 프롬프트 수정 | 특정 카테고리만 품질 급락 |
| **컨텍스트 코퍼스 변화** | RAG 문서가 추가/삭제/노후 → 검색 결과가 달라짐 | 특정 주제 답변만 부정확 |
| **질의 분포 이동 (distribution shift)** | 사용자가 골든셋에 없던 **새로운 종류**의 질문을 던짐 | 신규 기능·시즌 이슈 후 불만 ↑ |

핵심: **골든셋이 못 본 변화는 점수에 안 잡힌다.** 분포 이동은 특히 위험한데, 점수가 높은 이유가 "골든셋 질문엔 여전히 잘해서"일 뿐, 실사용 질문 분포와 어긋났기 때문이다.

### 8.2 근본 원인 귀속(Attribution) — 누가 범인인가

품질이 떨어졌을 때 RAG 파이프라인의 어느 컴포넌트가 원인인지 격리해야 한다. **한 번에 하나만 바꿔 A/B**(7장의 통계 기법 재사용)가 철칙이다 — 여러 개를 동시에 바꾸면 귀속이 불가능하다.

```
RAG 파이프라인 컴포넌트별 격리 진단:

질의 → [임베딩 모델] → [검색/청킹] → [리랭킹] → [LLM + 프롬프트] → 답변
          │                │            │             │
          ▼                ▼            ▼             ▼
   Context Recall↓?   Context        Precision↓?  Faithfulness↓?
   (검색 자체 실패)   Precision↓?   (순위 문제)   (생성 문제)
```

| 증상(지표) | 의심 컴포넌트 | 격리 실험 |
|---|---|---|
| Context Recall ↓ | 임베딩 모델 / 청킹 | 임베딩 모델만 롤백 후 동일 질의 |
| Context Precision ↓ | 리랭커 / top-k | 리랭킹만 끄고 측정 |
| Recall·Precision은 OK인데 Faithfulness ↓ | LLM / 프롬프트 | 컨텍스트 고정 후 모델·프롬프트만 교체 |
| 전부 OK인데 체감 ↓ | 질의 분포 이동 (8.1) | 프로덕션 로그를 골든셋과 대조 |

```python
# 컴포넌트 격리: 검색 결과를 "고정"하고 생성만 평가 → 생성 문제 분리
def isolate_generation(question, frozen_contexts, model_a, model_b):
    # 같은 컨텍스트를 주고 모델만 바꿔 Faithfulness 비교
    ans_a = model_a.generate(question, frozen_contexts)
    ans_b = model_b.generate(question, frozen_contexts)
    return judge_faithfulness(ans_a, frozen_contexts), \
           judge_faithfulness(ans_b, frozen_contexts)
```

### 8.3 회귀 추적 — 골든셋 노후화

가장 함정인 시나리오: **새 프롬프트가 Eval 점수는 더 높은데 사용자 불만은 늘었다.**

```
프롬프트 v2:  골든셋 Faithfulness 0.85 → 0.91 (개선!)  → 배포
프로덕션:     재질문율 12% → 18% (악화)
원인:         골든셋이 6개월 전 질문 분포 → 현재 실사용을 대표하지 못함
              v2는 "옛 질문"에 과적합, 신규 질문엔 더 나빠짐
```

이건 모델 회귀가 아니라 **측정 도구(골든셋)의 회귀**다. 해법은 골든셋을 **프로덕션 로그에서 상시 갱신**하는 것:

```python
# 프로덕션 로그 → 골든셋 갱신 파이프라인 (주기적 실행)
def refresh_golden_set(prod_logs, current_golden):
    # 1) 분포 이동 감지: 최근 질의를 클러스터링해 골든셋에 없는 클러스터 탐지
    new_clusters = find_uncovered_clusters(prod_logs, current_golden)

    # 2) 저품질 실사용 케이스 우선 샘플링 (불만/재질문/낮은 피드백)
    hard_cases = sample_by_signal(prod_logs,
                                  signals=["thumbs_down", "rephrase", "abandon"])

    # 3) 사람이 정답 라벨링 → 골든셋에 편입 (오래된/중복 케이스는 폐기)
    return current_golden.merge(label(new_clusters + hard_cases))
```

원칙: **골든셋은 살아있는 데이터셋**(5장의 "정기 갱신" 조건). "한 번 만든 Eval Suite"는 시간이 지나면 반드시 실사용과 괴리된다.

### 8.4 모니터링 전략 — 온라인 신호 × 오프라인 Eval 연결

오프라인 점수만 보면 8.1의 드리프트를 영원히 못 잡는다. **프로덕션의 암묵적 신호를 상시 수집**하고 오프라인 Eval과 묶어야 한다.

| 온라인 신호 | 의미 | 대응 오프라인 지표 |
|---|---|---|
| 👍/👎 사용자 피드백 | 직접적 품질 체감 | Faithfulness / Relevancy |
| **재질문율(rephrase)** | 첫 답이 부족 → 다시 물음 | Answer Relevancy |
| 세션 이탈/중단 | 답이 쓸모없어 포기 | 전반 품질 |
| 후속 정정 요청 | 사실 오류 의심 | Faithfulness |

```python
# 상시 온라인 Eval: 프로덕션 트래픽을 샘플링해 LLM-Judge 백그라운드 채점
def online_eval_sampler(production_traffic, sample_rate=0.02):
    for req in production_traffic:
        if random.random() < sample_rate:          # 2%만 샘플링(비용 통제)
            score = llm_judge(req.question, req.response, req.contexts)
            metrics.emit("online_faithfulness", score["faithfulness"],
                         tags={"model": req.model_version})
            if score["faithfulness"] < 0.7:          # 임계 미달이면
                quarantine_for_golden_set(req)       # 골든셋 후보로 격리(8.3)
```

운영 루프:

```
프로덕션 트래픽
   ├─ 온라인 신호 수집(피드백·재질문·이탈)  ──┐
   └─ 샘플링 LLM-Judge 상시 채점            ──┤
                                            ▼
            온라인 점수 하락 / 신호 악화 감지
                                            ▼
   골든셋에 없던 케이스 → 라벨링 → 골든셋 갱신(8.3)
                                            ▼
              컴포넌트 격리 진단(8.2)으로 범인 특정
                                            ▼
                    수정 → 오프라인 Eval(6장 CI) → 배포
```

> [!warning] 핵심 원칙
> **오프라인 Eval 통과는 "회귀하지 않았다"는 증명일 뿐, "프로덕션에서 좋다"는 증명이 아니다.** 온라인 신호가 오프라인 점수와 괴리되는 순간이 곧 골든셋을 갱신할 때다.

---

## 9. LLM-as-Judge 편향 주의

| 편향 유형 | 설명 | 완화 방법 |
|---|---|---|
| **Position Bias** | 첫 번째 응답을 선호 | A/B 순서 swap 후 평균 |
| **Length Bias** | 긴 응답을 더 좋다고 평가 | 길이 제한 지시 추가 |
| **Self-Enhancement** | 자기 자신(또는 같은 모델)의 응답 선호 | 다른 모델 Judge 사용 |
| **Verbosity** | 더 장황한 응답 선호 | 간결성을 명시적 평가 기준에 포함 |
| **Metric Gaming** | 형식 맞추기로 점수 올리기 가능 | 다양한 지표 조합 + 인간 검증 병행 |

```python
# Position bias 방지: 동일 케이스를 A-B, B-A 두 번 평가 후 비교
def unbiased_judge(q, resp_a, resp_b, judge):
    result1 = judge(q, resp_a, resp_b)   # A vs B
    result2 = judge(q, resp_b, resp_a)   # B vs A (순서 반전)

    if result1 == "A" and result2 == "B":  # 둘 다 A가 나음
        return "A"
    elif result1 == "B" and result2 == "A":  # 둘 다 B가 나음
        return "B"
    else:
        return "TIE"  # 불일치 → 무승부 처리
```

---

## 10. 관련
- [[LLM]] · [[RAG]] · [[Fine-Tuning]] · [[Agent-Harness]] · [[Prompt-Engineering]] · [[LLMOps]]
- [[Citation-Attribution]] — Faithfulness 오프라인 측정 ↔ 온라인 개별 인용 검증의 짝
- [[RAG-Failure-Debugging]] — 집계 점수(여기) → 단일 오답을 단계로 귀속하는 진단(context_recall vs faithfulness 읽기)

---

## 참고문헌

[1] S. Es et al., "RAGAS: Automated Evaluation of Retrieval Augmented Generation," 2023. arXiv:2309.15217
[2] D. Hendrycks et al., "Measuring Massive Multitask Language Understanding," *ICLR*, 2021. arXiv:2009.03300
[3] M. Chen et al., "Evaluating Large Language Models Trained on Code," 2021. arXiv:2107.03374
[4] L. Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena," *NeurIPS*, 2023. arXiv:2306.05685
