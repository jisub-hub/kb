---
tags:
  - ai
  - vlm
  - vision
  - multimodal
  - llm
created: 2026-06-16
---

# VLM (Vision Language Model)

> [!summary] 한 줄 요약
> **이미지·영상을 텍스트와 함께 이해하는 멀티모달 LLM**. 이미지를 토큰으로 변환해 언어 모델에 주입하는 방식이 주류. 문서 OCR, 차트 분석, 이미지 Q&A, 영상 이해 등 시각 기반 태스크를 처리한다.

---

## 1. VLM 아키텍처 — 이미지를 어떻게 처리하는가

```
이미지
  │
  ▼
[Vision Encoder]        ← 이미지 특징 추출 (보통 ViT 기반)
  │  이미지 패치 임베딩
  ▼
[Projection Layer]      ← 이미지 임베딩을 LLM 차원으로 변환 (MLP or Cross-Attention)
  │  "이미지 토큰"
  ▼
[LLM Backbone]          ← 텍스트 토큰 + 이미지 토큰을 함께 처리
  │
  ▼
텍스트 답변
```

### Vision Encoder — ViT (Vision Transformer)

ViT는 이미지를 고정 크기 패치로 분할해 각 패치를 토큰처럼 취급함으로써 순수 Transformer를 이미지 인식에 적용한 최초 연구다 [Dosovitskiy et al., 2021].

```
이미지 → 패치 분할 (14×14 또는 16×16 픽셀) → 각 패치를 토큰으로 임베딩

예: 336×336 이미지, 패치 크기 14×14
  → (336/14)² = 576개 패치 토큰

고해상도 이미지: 패치 수 증가 → 컨텍스트 토큰 수 증가 → 비용↑ 속도↓
```

| Vision Encoder | 사용 모델 | 특징 |
|---|---|---|
| CLIP ViT-L/14 [Radford et al., 2021] | LLaVA 1.5, 초기 모델 | 범용, 336px |
| CLIP ViT-L/14@336 | LLaVA 1.5 | 고해상도 |
| SigLIP | LLaVA-1.6, Gemma 3 | CLIP 개선, 다국어 |
| InternViT-6B | InternVL2 | 대형 ViT, 고품질 |
| DINOv2 | 일부 모델 | 공간 이해 강점 |

### Projection 방식

| 방식 | 설명 | 모델 |
|---|---|---|
| Linear / MLP | 단순 선형 변환 | LLaVA 초기 [Liu et al., 2023] |
| Q-Former | Cross-Attention으로 고정 수 쿼리 토큰 추출 | BLIP-2, InstructBLIP |
| Resampler | Perceiver 기반 토큰 압축 | Flamingo, Idefics |
| **직접 concat** | 패치 임베딩 그대로 LLM에 입력 | LLaVA-1.5 이후 표준 ⭐ |

---

## 2. 주요 VLM 모델

### 클라우드 API (상용)

| 모델 | 제공사 | 특징 |
|---|---|---|
| **Claude 3.5 Sonnet / Opus 4.8** | Anthropic | 문서 이해, 복잡 추론 최강 |
| GPT-4o | OpenAI | 이미지+오디오 통합, 빠름 |
| Gemini 2.5 Pro | Google | 긴 컨텍스트, 영상 이해 |

### 오픈소스 (로컬 실행 가능)

| 모델 | 파라미터 | Vision | 특징 |
|---|---|---|---|
| **Llama 3.2 Vision 11B** | 11B | SigLIP | Meta 공식, 로컬 ⭐ |
| **Llama 3.2 Vision 90B** | 90B | SigLIP | 고품질, M5 Max 128GB |
| **InternVL2-8B** | 8B | InternViT | 문서/차트 매우 강함 ⭐ |
| InternVL2-26B | 26B | InternViT | 고품질 |
| InternVL2-76B | 76B | InternViT-6B | 최고 품질 오픈소스 |
| **LLaVA-1.6 (34B)** | 34B | CLIP | 구조 단순, 범용 |
| **Qwen2.5-VL-7B** | 7B | ViT | 문서 OCR 강함 [Bai et al., 2023] |
| Qwen2.5-VL-72B | 72B | ViT | 최상위 오픈소스 급 |
| PaliGemma-3B | 3B | SigLIP | 경량, 파인튜닝 용이 |
| moondream2 | 1.8B | SigLIP | 초경량, 엣지 디바이스 |
| SmolVLM | 250M~2B | SigLIP | 최소형 VLM |

---

## 3. VLM 입력 처리 — 코드

### Ollama (로컬)

```bash
ollama pull llama3.2-vision:11b
ollama pull qwen2.5vl:7b

# CLI에서 이미지 포함 질의
ollama run llama3.2-vision:11b "이 이미지에서 무엇이 보이는가?" --image /path/to/image.jpg
```

### Python (Transformers)

```python
from transformers import AutoProcessor, AutoModelForVision2Seq
from PIL import Image
import torch

# LLaVA-1.6 / Llama 3.2 Vision
model_id = "meta-llama/Llama-3.2-11B-Vision-Instruct"
processor = AutoProcessor.from_pretrained(model_id)
model = AutoModelForVision2Seq.from_pretrained(
    model_id,
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

image = Image.open("chart.png")

messages = [
    {
        "role": "user",
        "content": [
            {"type": "image"},
            {"type": "text", "text": "이 차트에서 2023년 매출 증가율을 알려줘."}
        ]
    }
]

# 프로세서가 이미지 → 패치 토큰 변환
input_text = processor.apply_chat_template(messages, add_generation_prompt=True)
inputs = processor(image, input_text, return_tensors="pt").to(model.device)

output = model.generate(**inputs, max_new_tokens=512)
print(processor.decode(output[0], skip_special_tokens=True))
```

### Claude API (이미지 포함)

```python
import anthropic
import base64

client = anthropic.Anthropic()

# 이미지를 base64로 인코딩
with open("invoice.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": image_data,
                    },
                },
                {
                    "type": "text",
                    "text": "이 인보이스에서 총 금액, 발행일, 수신인을 JSON으로 추출해줘."
                }
            ],
        }
    ],
)
print(response.content[0].text)

# URL 방식 (공개 이미지)
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=512,
    messages=[{"role": "user", "content": [
        {"type": "image", "source": {"type": "url", "url": "https://..."}},
        {"type": "text", "text": "무엇이 보이는가?"}
    ]}]
)
```

### InternVL2 (문서/차트 특화)

```python
# InternVL2는 문서 OCR, 차트 분석에서 오픈소스 최강
from transformers import InternVLChatModel, CLIPImageProcessor
import torch

model = InternVLChatModel.from_pretrained(
    "OpenGVLab/InternVL2-8B",
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

# 다중 이미지 지원
response = model.chat(
    processor, images=[img1, img2],
    question="두 문서의 차이점은?",
    generation_config=dict(max_new_tokens=512)
)
```

---

## 4. 고해상도 처리 전략

단순히 이미지를 리사이즈하면 세부 정보 손실 → **타일링(Tiling)** 전략 도입.

```
[LLaVA-1.6 / InternVL2 타일링]

원본 이미지 (고해상도)
    │
    ├── 전체 축소 썸네일 (1개)   ← 전체 구조 파악
    ├── 좌상단 타일 (2개)
    ├── 우상단 타일 (2개)
    └── ... (N개 타일)

→ 모든 타일을 독립 패치로 인코딩 후 concat
→ 텍스트 인식, 세밀한 차트 분석 가능
```

| 전략 | 토큰 수 | 품질 |
|---|---|---|
| 단순 리사이즈 (224px) | ~256 토큰 | 낮음 |
| 단순 리사이즈 (336px) | ~576 토큰 | 중간 |
| 타일링 (2×2) | ~2304 토큰 | 높음 |
| 동적 타일링 (최대 12타일) | ~6912 토큰 | 최고 |

---

## 5. VLM 주요 태스크

| 태스크 | 설명 | 권장 모델 |
|---|---|---|
| **이미지 Q&A** | 이미지에 대한 자연어 질문 | Claude, Llama 3.2 Vision |
| **OCR / 문서 추출** | 텍스트, 표, 양식 추출 | InternVL2, Qwen2.5-VL ⭐ |
| **차트 / 그래프 분석** | 데이터 수치 읽기, 해석 | InternVL2, Claude |
| **이미지 캡셔닝** | 이미지 설명 생성 | 대부분의 VLM |
| **이미지 분류** | 카테고리 판별 | PaliGemma, moondream |
| **다중 이미지 비교** | 여러 이미지 간 차이 분석 | InternVL2, Claude |
| **영상 이해** | 비디오 프레임 분석 | Gemini, Qwen2.5-VL |
| **화면 이해 (UI)** | 스크린샷 분석, 자동화 | Claude, GPT-4o |

---

## 6. 시각 환각 (Visual Hallucination)

VLM이 **이미지에 없는 객체·텍스트·정보를 생성**하는 현상. 텍스트 LLM의 환각과 다른 특성.

```
[텍스트 환각]  없는 사실을 그럴듯하게 생성
[시각 환각]    이미지에 없는 것을 "보였다"고 주장
              또는 이미지의 텍스트를 잘못 읽음 (OCR 오류)
              또는 이미지와 무관한 지식으로 답변
```

### 주요 유형

| 유형 | 예시 | 원인 |
|---|---|---|
| 객체 환각 | 이미지에 고양이 없는데 "고양이가 있다" | 언어 편향, 컨텍스트 과의존 |
| OCR 오류 | "ABC"를 "A8C"로 읽음 | 저해상도, 폰트 복잡성 |
| 수량 오류 | "사람 3명"인데 "5명" | 작은 객체, 겹침 |
| 관계 오류 | 위치/방향 관계 혼동 | 공간 추론 한계 |
| 텍스트 편향 | 이미지 무시하고 학습 데이터 기반 답변 | 언어 모델 우선 |

### 완화 방법

```python
# 1. 근거 요구 프롬프트 — 이미지에서 본 증거를 인용하게 강제
prompt = """이미지를 분석하라.
답변 시 반드시 이미지의 어느 부분에서 확인했는지 명시하라.
이미지에서 확인할 수 없는 정보는 "이미지에서 확인 불가"라고 답하라."""

# 2. 불확실성 표현 요구
prompt = "이미지에서 확실하지 않은 부분은 [불확실] 표시를 붙여라."

# 3. 고해상도 입력 — 저해상도 이미지는 환각↑
# 최소 672×672px 이상 권장, 문서 OCR은 원본 해상도 유지
```

### OCR 신뢰도

```
일반 VLM OCR 적합:  깔끔한 인쇄 텍스트, 큰 폰트
주의 필요:          손글씨, 작은 텍스트, 기울어진 텍스트
전용 OCR 권장:      금융 문서, 계약서 (Tesseract, PaddleOCR, Azure DI)
```

---

## 7. 비디오 VLM

영상을 **프레임 샘플링 → 각 프레임을 이미지 토큰으로 처리**.

```
비디오 (30fps, 1분)
    │ 프레임 샘플링 (1fps 또는 scene change 기반)
    ↓
[frame₁, frame₂, ..., frame₆₀]  각 ~576 토큰
    │
총 토큰: 60 × 576 = 34,560 토큰  ← 컨텍스트 비용 주의
```

| 모델 | 비디오 지원 | 최대 길이 | 특징 |
|---|---|---|---|
| **Gemini 2.5 Pro** | ✅ 네이티브 | 2시간+ | 긴 영상 이해 최강 |
| GPT-4o | ✅ 프레임 | 제한적 | API로 프레임 전송 |
| Qwen2.5-VL-72B | ✅ | ~20분 | 오픈소스 최강 |
| InternVL2-76B | ⚠️ 제한적 | 짧은 클립 | |

```python
# 비디오 프레임 샘플링 (OpenCV)
import cv2

def sample_frames(video_path: str, fps: int = 1) -> list:
    cap = cv2.VideoCapture(video_path)
    video_fps = cap.get(cv2.CAP_PROP_FPS)
    interval = int(video_fps / fps)
    frames = []
    i = 0
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret: break
        if i % interval == 0:
            frames.append(frame)
        i += 1
    return frames
```

---

## 8. VLM 파인튜닝

```python
# PaliGemma (가장 파인튜닝하기 쉬운 경량 VLM)
# 태스크 특화 VLM 만들기 — 의료 이미지, 위성 사진 분류 등

from transformers import PaliGemmaForConditionalGeneration, PaliGemmaProcessor
from peft import LoraConfig, get_peft_model

model = PaliGemmaForConditionalGeneration.from_pretrained(
    "google/paligemma-3b-pt-224",
    torch_dtype=torch.bfloat16
)

lora_config = LoraConfig(
    r=16, lora_alpha=32,
    target_modules=["q_proj", "v_proj"],  # LLM 부분만 LoRA
    task_type="CAUSAL_LM"
)
model = get_peft_model(model)
```

---

## 7. VLM vs 텍스트 LLM 비교

| 항목 | 텍스트 LLM | VLM |
|---|---|---|
| 입력 | 텍스트만 | 텍스트 + 이미지(+영상) |
| 토큰 비용 | 텍스트 토큰만 | 이미지 토큰 추가 (수백~수천) |
| 추론 속도 | 빠름 | 이미지 인코딩 오버헤드 |
| 메모리 | 상대적으로 작음 | Vision Encoder 추가 |
| 로컬 실행 | 용이 | Vision Encoder 메모리 추가 |

---

## 8. 관련
- [[LLM]] · [[Transformer-Architecture]] · [[Open-Source-Models]] · [[Inference-Optimization]]
- [[Multimodal-RAG]] — VLM을 검색·생성에 쓰는 시각 문서 RAG(ColPali류)
- [[Apple-M5-Max]] · [[DGX-Spark]]

---

## 참고문헌

[1] A. Dosovitskiy et al., "An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale," *ICLR*, 2021. arXiv:2010.11929
[2] A. Radford et al., "Learning Transferable Visual Models From Natural Language Supervision," *ICML*, 2021. arXiv:2103.00020
[3] H. Liu et al., "Visual Instruction Tuning," *NeurIPS*, 2023. arXiv:2304.08485
[4] J. Bai et al., "Qwen-VL: A Versatile Vision-Language Model for Understanding, Localization, Text Reading, and Beyond," 2023. arXiv:2308.12966
