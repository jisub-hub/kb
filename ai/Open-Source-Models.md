---
tags:
  - ai
  - llm
  - open-source
  - hermes
  - llama
  - mistral
  - models
created: 2026-06-16
---

# Open-Source LLM Models

> [!summary] 한 줄 요약
> 로컬 실행 가능한 오픈소스 LLM 모델들. Llama 3, Mistral, Hermes, Qwen 등 주요 계열 특성과 선택 기준 정리. [[Apple-M5-Max]], [[DGX-Spark]]에서 mlx-lm/Ollama로 실행 가능.

---

## 1. 모델 패밀리 개요

```
Meta (Llama)
  └── Llama 3.1: 8B / 70B / 405B — 현재 오픈소스 기준점
  └── Llama 3.2: 1B / 3B / 11B(Vision) / 90B(Vision)
  └── Llama 3.3: 70B (Llama 3.1 70B 대비 향상)

Mistral AI
  └── Mistral 7B — 소형 고성능의 기준
  └── Mixtral 8×7B — MoE, 47B 파라미터 / 13B 활성
  └── Mixtral 8×22B — 대형 MoE
  └── Mistral NeMo 12B — 128K 컨텍스트

NousResearch (Hermes)
  └── Hermes-3-Llama-3.1-8B
  └── Hermes-3-Llama-3.1-70B
  └── Nous-Hermes-2-Mixtral-8x7B

Alibaba (Qwen)
  └── Qwen2.5: 0.5B~72B, 코딩/수학 특화
  └── QwQ-32B — 추론 특화 (DeepSeek-R1 류)

DeepSeek
  └── DeepSeek-V3 — Mixture of Experts
  └── DeepSeek-R1 — 추론 특화 (chain-of-thought)

Google
  └── Gemma 2: 2B / 9B / 27B

Microsoft
  └── Phi-4: 14B — 소형 고효율
```

---

## 2. Hermes — NousResearch

> **Hermes는 기존 Llama/Mistral 모델을 SFT+DPO로 파인튜닝한 모델**. 지시 따르기, Function Calling, JSON 출력에서 특히 강점.

### 특징

| 능력 | 수준 | 비고 |
|---|---|---|
| **Function Calling / Tool Use** | ⭐⭐⭐ 매우 강함 | Hermes의 핵심 강점 |
| **JSON / 구조화 출력** | ⭐⭐⭐ 매우 강함 | 형식 준수율 높음 |
| 지시 따르기 | ⭐⭐⭐ | 복잡한 시스템 프롬프트 처리 |
| 다국어 | ⭐⭐ | 한국어 기본 지원 |
| 코딩 | ⭐⭐⭐ | 코드 생성 우수 |
| 추론 | ⭐⭐ | 70B에서 강함 |

### Hermes 3 — 핵심 버전

```bash
# Ollama로 실행
ollama pull hermes3:8b
ollama pull hermes3:70b
ollama run hermes3:8b

# mlx-lm (Apple Silicon)
python -m mlx_lm.generate \
  --model mlx-community/Hermes-3-Llama-3.1-8B-4bit \
  --prompt "..."
```

### Hermes Function Calling 포맷

Hermes는 `<tool_call>` XML 태그 기반의 함수 호출 포맷을 사용:

```
[시스템 프롬프트에 툴 정의]
<tools>
[{"name": "get_weather", "description": "...", "parameters": {...}}]
</tools>

[어시스턴트 응답]
<tool_call>
{"name": "get_weather", "arguments": {"city": "Seoul"}}
</tool_call>

[툴 결과 주입]
<tool_response>
{"temperature": 22, "weather": "맑음"}
</tool_response>

[최종 응답]
서울의 현재 날씨는 맑고 22°C입니다.
```

```python
# Python 예시 (llama.cpp / transformers)
system_prompt = """You are a helpful assistant with access to tools.
<tools>
[
  {
    "name": "search_products",
    "description": "상품을 검색한다",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {"type": "string"},
        "category": {"type": "string"}
      },
      "required": ["query"]
    }
  }
]
</tools>"""
```

### Hermes 모델 라인업

| 모델 | 기반 | 파라미터 | 특징 |
|---|---|---|---|
| **Hermes-3-Llama-3.1-8B** | Llama 3.1 8B | 8B | 경량, 빠른 응답, 로컬 ⭐ |
| **Hermes-3-Llama-3.1-70B** | Llama 3.1 70B | 70B | 고품질, 복잡한 에이전트 |
| Hermes-3-Llama-3.1-405B | Llama 3.1 405B | 405B | 최고 품질, 클라우드 필요 |
| Nous-Hermes-2-Mixtral-8x7B | Mixtral 8×7B | 47B(MoE) | 빠른 MoE, 균형 |
| Nous-Hermes-2-Yi-34B | Yi-34B | 34B | 다국어 강점 |

---

## 3. Llama 3 / 3.1 / 3.3

Meta의 오픈소스 LLM. 현재 오픈소스 생태계의 기준 모델.

| 버전 | 파라미터 | 컨텍스트 | 특징 |
|---|---|---|---|
| **Llama 3.1 8B** | 8B | 128K | 로컬 실용 모델 |
| **Llama 3.1 70B** | 70B | 128K | M5 Max / DGX Spark 추천 ⭐ |
| Llama 3.1 405B | 405B | 128K | 최고 품질, 다중 GPU |
| Llama 3.2 3B | 3B | 128K | 온디바이스 |
| Llama 3.2 90B Vision | 90B | 128K | 멀티모달 (이미지 이해) |
| **Llama 3.3 70B** | 70B | 128K | Llama 3.1 70B 대비 향상 |

```bash
ollama pull llama3.1:70b
ollama pull llama3.3:70b   # 더 나은 성능
```

---

## 4. Mistral / Mixtral

프랑스 Mistral AI. 효율성이 강점.

| 모델 | 파라미터 | 특징 |
|---|---|---|
| Mistral 7B v0.3 | 7B | 소형 고성능 기준 |
| **Mixtral 8×7B** | 47B(13B 활성) | MoE, 빠름 |
| Mistral NeMo 12B | 12B | 128K 컨텍스트, 균형 |
| Mistral Large 2 | 123B | 고품질, API 제공 |

---

## 5. Qwen 2.5 — Alibaba

코딩·수학에서 같은 크기 대비 강점.

| 모델 | 파라미터 | 특징 |
|---|---|---|
| Qwen2.5-7B-Instruct | 7B | 한국어 지원 우수 |
| **Qwen2.5-32B-Instruct** | 32B | 128GB 내 고품질 |
| Qwen2.5-72B-Instruct | 72B | 최고 품질 |
| Qwen2.5-Coder-32B | 32B | 코딩 특화 |
| **QwQ-32B** | 32B | 추론(CoT) 특화 ⭐ |

---

## 6. DeepSeek

중국 DeepSeek. 추론 능력과 비용 효율 주목.

| 모델 | 특징 |
|---|---|
| DeepSeek-V3 | 671B MoE, GPT-4급 품질 |
| **DeepSeek-R1** | Chain-of-Thought 추론 특화, 수학/코딩 최강 |
| DeepSeek-R1-Distill-8B | R1의 지식을 8B에 증류, 로컬 추론 |
| DeepSeek-R1-Distill-70B | R1 증류 70B |

---

## 7. 모델 선택 가이드

| 사용 목적 | 권장 모델 | 이유 |
|---|---|---|
| **로컬 에이전트 / Function Calling** | Hermes-3 8B/70B | 툴 호출 정확도 최고 |
| **일반 채팅 / 지시 따르기** | Llama 3.3 70B | 균형 잡힌 성능 |
| **코딩** | Qwen2.5-Coder-32B | 코딩 특화 |
| **수학 / 논리 추론** | QwQ-32B, DeepSeek-R1 | CoT 추론 특화 |
| **빠른 응답 (8B급)** | Llama 3.1 8B, Mistral 7B | 속도 우선 |
| **한국어** | Qwen2.5, bge-m3 임베딩 | 한국어 학습 데이터 |
| **멀티모달** | Llama 3.2 90B Vision | 이미지 이해 |
| **대규모 배치 (프로덕션)** | vLLM + Llama 3.1 70B AWQ | 처리량 최대화 |

---

## 8. mlx-lm 커뮤니티 모델 (Apple Silicon 최적화)

`mlx-community` HuggingFace 조직에서 MLX 포맷으로 변환된 모델 제공:

```bash
# 자주 쓰이는 mlx-community 모델
mlx-community/Meta-Llama-3.1-8B-Instruct-4bit
mlx-community/Meta-Llama-3.1-70B-Instruct-4bit
mlx-community/Hermes-3-Llama-3.1-8B-4bit
mlx-community/Hermes-3-Llama-3.1-70B-4bit
mlx-community/Qwen2.5-32B-Instruct-4bit
mlx-community/QwQ-32B-4bit
mlx-community/DeepSeek-R1-Distill-70B-4bit
```

---

## 9. 관련
- [[LLM]] · [[Inference-Optimization]] · [[Fine-Tuning]]
- [[Apple-M5-Max]] · [[DGX-Spark]]
