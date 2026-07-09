---
tags:
  - ai
  - sllm
  - small-llm
  - edge
  - on-device
  - phi
  - gemma
created: 2026-06-16
---

# SLLM (Small Language Model)

> [!summary] 한 줄 요약
> **1B~14B 파라미터급 소형 언어 모델**. 엣지 디바이스·온디바이스·저지연·저비용 환경에 특화. "작지만 잘 학습된" 모델은 특정 태스크에서 대형 모델과 경쟁하거나 능가한다. 로컬 실행, 프라이버시, 비용 절감이 핵심 이유.

---

## 1. 왜 SLLM인가

```
LLM 선택의 트레이드오프:

크기      성능    속도    비용    프라이버시    엣지 실행
────────────────────────────────────────────────────
GPT-4o    ⭐⭐⭐⭐⭐  ⭐⭐⭐   $$$$   ❌ (클라우드)  ❌
Llama 70B ⭐⭐⭐⭐   ⭐⭐    $$$    ✅ 로컬       △ (고사양)
Phi-4 14B ⭐⭐⭐    ⭐⭐⭐⭐  $      ✅ 로컬       ✅
Gemma 2B  ⭐⭐     ⭐⭐⭐⭐⭐ $      ✅ 로컬       ✅ (모바일도)
```

### SLLM이 유리한 상황

- **온디바이스/엣지**: 스마트폰, IoT, 오프라인 환경
- **저지연 필요**: 실시간 자동완성, 코드 제안
- **비용 민감**: API 비용 절감, 대용량 배치 처리
- **프라이버시**: 사내 문서, 개인 데이터 로컬 처리
- **단순 반복 태스크**: 분류, 추출, 형식 변환 — 대형 모델 불필요

---

## 2. 주요 SLLM 모델

### Microsoft Phi (품질 중심 소형 모델)

| 모델 | 파라미터 | 특징 |
|---|---|---|
| **Phi-4** | 14B | 수학·추론·코딩 14B 최강 ⭐ |
| Phi-4 Mini | 3.8B | 모바일/엣지 가능 |
| Phi-3.5 Mini | 3.8B | 128K 컨텍스트, 저전력 |
| Phi-3.5 MoE | 16×3.8B (6.6B 활성) | 빠른 MoE |

```
Phi 모델의 철학:
"적은 파라미터에 고품질 데이터를 집중 학습"
  → 교과서 품질 데이터 (합성 데이터 포함)
  → 중복/저품질 데이터 제거
  → 같은 파라미터 대비 월등한 성능
```

Phi-4는 사전학습 전 과정에서 합성 데이터를 전략적으로 활용하며, 교사 모델인 GPT-4o를 STEM 특화 QA에서 능가함으로써 단순 증류를 넘어서는 데이터 중심 학습법의 가능성을 보였다 [Abdin et al., 2024].

### Google Gemma 2 (경량 고효율)

| 모델 | 파라미터 | 특징 |
|---|---|---|
| **Gemma 2 2B** | 2B | 모바일, 매우 빠름 ⭐ |
| **Gemma 2 9B** | 9B | 10B 미만 최고 품질 |
| Gemma 2 27B | 27B | 중형 고품질 |
| **Gemma 3 1B** | 1B | 온디바이스, 멀티모달 |
| Gemma 3 4B | 4B | 엣지, 이미지 이해 |
| Gemma 3 12B | 12B | 균형, 멀티모달 |

```
Gemma 2의 기술:
  - Knowledge Distillation (대형 교사 모델로 학습)
  - Sliding Window Attention (로컬 + 글로벌 attention 교차)
  - Logit Soft-Capping (수치 안정성)
```

### Meta Llama 3.2 (소형 라인)

| 모델 | 파라미터 | 특징 |
|---|---|---|
| **Llama 3.2 1B** | 1B | 온디바이스, 초경량 |
| **Llama 3.2 3B** | 3B | 엣지, 빠른 응답 |

### Alibaba Qwen 2.5 소형

| 모델 | 파라미터 | 특징 |
|---|---|---|
| Qwen2.5-0.5B | 0.5B | 극경량 |
| Qwen2.5-1.5B | 1.5B | 한국어 지원 소형 |
| **Qwen2.5-3B** | 3B | 균형, 한국어 ⭐ |
| Qwen2.5-7B | 7B | 7B급 최강 수준 |

### 특수 목적 소형 모델

| 모델 | 파라미터 | 특화 |
|---|---|---|
| **DeepSeek-R1-Distill-7B** | 7B | 추론(CoT), R1 증류 |
| **DeepSeek-R1-Distill-1.5B** | 1.5B | 초경량 추론 |
| SmolLM2-1.7B | 1.7B | 엣지, HuggingFace 경량 |
| **moondream2** | 1.8B | VLM, 엣지 이미지 |
| Qwen2.5-Coder-1.5B | 1.5B | 코딩, IDE 플러그인용 |

---

## 3. SLLM 성능의 핵심 — 학습 데이터 품질

```
같은 파라미터라도 학습 데이터에 따라 성능이 크게 달라진다.

Phi-4 14B의 비결:
  1. 합성 데이터 (Synthetic Data)
     → GPT-4o로 고품질 Q&A, 단계별 추론 생성
  2. 교과서 품질 필터링
     → 웹 크롤링 데이터에서 고품질만 선별
  3. 수학/코딩 특화
     → 논리적 추론이 필요한 데이터에 집중

결과: Phi-4 14B가 MATH-500, GPQA 등 **수학·추론 특화 벤치마크**에서 Llama 3.1 70B를 능가.
      단, 일반 지식·다국어·긴 컨텍스트 태스크에서는 70B가 우세.
```

---

## 4. Knowledge Distillation (지식 증류)

**대형 모델(교사)의 지식을 소형 모델(학생)에 전달** [Hinton et al., 2015].

```
[교사 모델 - Teacher]          [학생 모델 - Student]
  Llama 3.1 70B                 Llama 3.2 3B
       │                              │
       │ Soft labels                  │ 학습
       │ (확률 분포)                   │
       └──────────────────────────────►│
                                      
일반 SFT:  학생이 정답(0 or 1)만 학습
Distill:   학생이 교사의 확률 분포 전체를 학습
           → "왜 이 답인가"의 미묘한 확률 정보까지 학습
```

```
손실 함수 (Distillation):
  L = α × L_CE(학생, 정답) + (1-α) × L_KL(학생, 교사 분포)
  
  α = 0.5 (정답 학습 50%, 교사 지식 50%)
  Temperature = 4~10 (교사 출력을 부드럽게 만들어 정보량↑)
```

DeepSeek-R1-Distill 시리즈가 대표적:
- DeepSeek-R1 (671B MoE)의 추론 능력을 1.5B~70B에 증류
- 7B 모델이 GPT-4o 수준 수학 추론 달성

---

## 5. 온디바이스 / 엣지 배포

### Apple Neural Engine (ANE)

```
CoreML로 변환 → ANE에서 실행
  → Neural Engine 16코어 ~40+ TOPS
  → 배터리 소모 최소화 (GPU 대비 10배 전력 효율)

적합: 분류·임베딩·소형 언어 모델 (1~3B)
```

```python
# CoreML 변환 (소형 모델)
import coremltools as ct
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("microsoft/phi-4-mini")
# → ct.convert()로 .mlpackage 생성 → iOS/macOS 앱에 번들
```

### GGUF + llama.cpp (크로스 플랫폼)

```bash
# Raspberry Pi, 저사양 PC, Mac, Windows 모두 동작
# Q4_K_M 양자화로 3B 모델 ~2GB

ollama pull phi4:latest          # 14B
ollama pull gemma2:2b            # 2B
ollama pull llama3.2:1b          # 1B

# 직접 llama.cpp
./llama-cli -m phi-4-mini-Q4_K_M.gguf -p "안녕하세요" --ctx-size 4096
```

### Android / iOS 배포

| 프레임워크 | 플랫폼 | 지원 모델 |
|---|---|---|
| **CoreML / ANE** | iOS, macOS | Gemma 2B, Phi-4 Mini (변환 필요) |
| **MediaPipe LLM** | Android, iOS | Gemma 2B, Phi-3 Mini |
| **MLC LLM** | Android, iOS, Web | Llama, Phi, Qwen |
| **llama.cpp Android** | Android (NDK) | 모든 GGUF |
| **ExecuTorch** | iOS, Android | PyTorch 모델 |

---

## 6. 벤치마크 — 소형 모델 비교

```
수학 (MATH-500):
  Phi-4 14B:          80.4%  ← 14B 최강
  Gemma 2 9B:         68.6%
  Llama 3.1 8B:       47.6%
  Gemma 2 2B:         43.7%

코딩 (HumanEval):
  Phi-4 14B:          82.6%
  Qwen2.5-7B:         75.2%
  Gemma 2 9B:         71.5%
  Llama 3.2 3B:       41.8%

추론 (ARC-C):
  Phi-4 14B:          92.8%
  Gemma 2 9B:         88.9%
  Llama 3.2 3B:       78.0%
  Gemma 2 2B:         74.7%
```

---

## 7. 용도별 SLLM 선택 가이드

| 사용 목적 | 권장 모델 | 이유 |
|---|---|---|
| **로컬 범용 (메모리 여유)** | Phi-4 14B | 14B 최고 품질 |
| **로컬 범용 (빠른 응답)** | Gemma 2 9B | 속도·품질 균형 |
| **온디바이스 / 모바일** | Gemma 3 4B, Phi-4 Mini | 저전력, 소형 |
| **초경량 (1~2B)** | Gemma 2 2B, Llama 3.2 1B | 라즈베리 파이, 배터리 |
| **한국어 소형** | Qwen2.5-3B, Qwen2.5-7B | 한국어 데이터 풍부 |
| **수학·코딩 소형** | Phi-4, DeepSeek-R1-Distill-7B | 추론 특화 |
| **VLM 소형** | moondream2, Gemma 3 4B | 이미지 이해 |
| **RAG 임베딩** | all-MiniLM-L6-v2, bge-m3 | 임베딩 특화 별도 |

---

## 8. SLLM 한계 및 실전 주의사항

### 성능 한계

- **복잡한 추론**: 다단계 수학·논리는 대형 모델 대비 오류율 높음. CoT 프롬프트로 일부 보완 가능
- **긴 컨텍스트 품질 저하**: 대부분의 소형 모델은 32K~128K를 지원하나, **실효 컨텍스트**는 8K~16K 내외. 긴 입력 중반부 정보를 놓치는 "Lost in the Middle" 현상이 대형 모델보다 더 심하게 나타남 [Liu et al., 2024]
- **환각 증가**: 파라미터가 적을수록 사실 저장 용량 부족 → [[RAG]]와 함께 사용 권장
- **다국어 편차**: 한국어는 Qwen 계열이 상대적으로 강하나 영어 대비 여전히 낮음

### 구조화 출력 신뢰도

소형 모델은 **JSON 스키마 준수 실패율**이 대형 모델보다 높다.

```python
# ❌ 소형 모델에서 프롬프트만으로 JSON 강제 — 실패 잦음
prompt = "다음을 JSON으로 답하라: ..."

# ✅ Ollama structured output (json_schema 강제)
import ollama
response = ollama.chat(
    model="phi4:latest",
    messages=[{"role": "user", "content": "사용자 정보 추출: 홍길동, 30세, 서울"}],
    format={
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"},
            "city": {"type": "string"}
        },
        "required": ["name", "age", "city"]
    }
)

# ✅ Outlines / guidance 라이브러리로 문법 강제 (llama.cpp 기반)
# → 토큰 샘플링 단계에서 스키마 위반 토큰을 마스킹
```

### 언제 SLLM 대신 대형 모델을 써야 하는가

```
SLLM으로 충분:
  ✅ 단순 분류 / 감성 분석
  ✅ 정형화된 정보 추출 (JSON 스키마가 있을 때)
  ✅ 짧은 문서 요약 (~4K 토큰 이내)
  ✅ 코드 자동완성 (IDE 플러그인)
  ✅ 온디바이스 / 오프라인

대형 모델 필요:
  ❌ 긴 문서 분석 (계약서 전체 검토)
  ❌ 복잡한 멀티스텝 에이전트
  ❌ 창의적 긴 형식 글쓰기
  ❌ 고품질 번역 / 다국어
  ❌ 도메인 지식 깊이 필요한 QA
```

---

## 9. 관련
- [[LLM]] · [[Open-Source-Models]] · [[Inference-Optimization]] · [[Fine-Tuning]]
- [[VLM]] · [[Apple-M5-Max]] · [[DGX-Spark]]

---

## 참고문헌

[1] G. Hinton, O. Vinyals & J. Dean, "Distilling the Knowledge in a Neural Network," *NeurIPS Deep Learning Workshop*, 2015. arXiv:1503.02531
[2] M. Abdin et al., "Phi-4 Technical Report," 2024. arXiv:2412.08905
[3] N. F. Liu et al., "Lost in the Middle: How Language Models Use Long Contexts," *Transactions of the Association for Computational Linguistics*, vol. 12, pp. 157–173, 2024. arXiv:2307.03172
