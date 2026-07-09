---
tags:
  - hardware
  - apple-silicon
  - inference
  - llm
  - bandwidth
  - npu
created: 2026-06-17
---

# Apple Silicon LLM 추론 — GPU Core · NPU · 대역폭의 역할

> [!summary] 한 줄 요약
> Mac에서 LLM **토큰 생성 속도(decode)는 통합 메모리 대역폭이 결정**한다. GPU 코어는 prefill·배칭·이미지생성 같은 *compute-bound* 작업에만 기여하고, NPU(Neural Engine)는 LLM 추론에 거의 쓰이지 않는다. "코어·NPU가 많으면 LLM이 빠르다"는 흔한 오해다.

---

## 1. 핵심 원리 — decode는 memory-bound

LLM 추론의 토큰 생성(decode)은 매 토큰마다 모델 가중치를 다시 읽는다. 연산량 대비 데이터 이동이 압도적이라(산술 강도 ≈ 1 FLOP/byte) **메모리 대역폭이 천장**이다. → [[../../ai/LLM-Inference-Bandwidth]]

```
decode TPS ≈ 통합 메모리 대역폭 / 모델 크기
  → GPU 코어 수, NPU 코어 수는 이 식에 들어가지 않는다
  → 코어를 늘려도 토큰 생성 속도는 거의 그대로
```

이것이 Apple Silicon에도 그대로 적용된다. UMA(통합 메모리)라 CPU·GPU·NPU가 같은 메모리 풀을 공유하므로, **어느 유닛을 쓰든 대역폭이라는 동일한 천장**에 묶인다.

---

## 2. GPU 코어가 기여하는 곳 (compute-bound 작업)

GPU 코어가 많으면 **"행렬-행렬 곱"이 많은 작업**이 빨라진다.

| 작업 | 코어 영향 | 설명 |
|------|----------|------|
| **Prefill** (프롬프트 처리) | ⭐⭐ | 긴 프롬프트·RAG 컨텍스트 → 첫 토큰까지(TTFT) 단축 |
| **배칭/다수 동시 요청** | ⭐⭐ | compute 여력으로 동시 처리량 ↑ |
| **이미지/비디오 생성** | ⭐⭐⭐ | Stable Diffusion 등, 완전 compute-bound → 코어가 직접 속도 결정 |
| **VLM 비전 인코더(ViT)** | ⭐⭐ | 이미지 패치 처리 |
| **학습/파인튜닝** | ⭐⭐ | compute-bound (단 Mac은 학습엔 약함) |
| **Decode** (토큰 생성) | ✕ | memory-bound → 코어 무관 |

```
이미지 생성    = 코어가 왕 (연산량이 병목)
LLM 채팅(decode) = 대역폭이 왕 (코어 무관)
LLM 긴 프롬프트(prefill) = 코어가 거듦
```

---

## 3. Apple 특유의 함정 — 코어 수와 대역폭의 관계

"코어 많은 Mac이 LLM 빠르더라"는 *착시*가 흔하다. 실제로는 같이 따라온 대역폭 덕분이다.

```
케이스 A: 칩 등급이 오름 (Pro → Max → Ultra)
  → 코어 + 대역폭이 함께 증가 → decode 빨라짐
  → 빨라진 진짜 이유는 "대역폭"이지 코어가 아니다

케이스 B: 같은 칩의 코어 옵션 차이 (예: Max 32코어 vs 40코어, 대역폭 동일)
  → decode TPS 거의 동일 (대역폭 같으니까)
  → prefill만 코어 많은 쪽이 빠름
```

### 칩 등급별 대역폭 (대략, 세대마다 다름)

| 등급 | 통합 메모리 대역폭 | decode 적합도 |
|------|------------------|--------------|
| _ (base) | ~100 GB/s | 소형 모델만 |
| _ Pro | ~200~270 GB/s | 중소형 |
| _ Max | ~400~600 GB/s | 중대형, 실용적 |
| _ Ultra | ~800 GB/s ~ 1.1 TB/s | 대형(70B급) |

> **같은 대역폭에서 코어만 더 많은 모델**을 골라도 채팅 속도(decode)는 거의 그대로다. 추론이 목적이면 코어에 쓸 돈을 대역폭(상위 등급)·용량에 쓰는 게 낫다.

---

## 4. NPU (Neural Engine, ANE)

별도의 저전력 추론 가속기다 (M4 기준 ~38 TOPS). CoreML로 접근한다.

**강점**: 와트당 효율이 압도적 — 작고 고정된 모델을 극저전력으로 상시 실행.

```
ANE가 실제 쓰이는 곳:
  얼굴 인식, 사진 분석, 음성 인식, 작은 비전/분류 모델 (온디바이스 ML)
  Apple 시스템 기능 (Siri, Apple Intelligence 일부)
```

**그러나 LLM 추론엔 거의 기여하지 않는다:**

```
① 주요 LLM 런타임(llama.cpp·MLX·Ollama)이 ANE가 아닌 GPU(Metal)를 사용
② ANE는 대용량 가중치 스트리밍·큰 KV cache 처리에 부적합
③ 결정적 — ANE도 "같은 통합 메모리·같은 대역폭"을 공유
   → decode TPS 천장은 GPU와 동일 (대역폭이 안 바뀜)
```

> NPU가 있어도 70B를 더 빨리 못 돌린다. NPU의 가치는 "속도"가 아니라 **"극저전력으로 작은 모델 상시 추론"**이다. LLM 구매 결정 요소가 아니다.

---

## 5. 구성요소별 역할 정리

| 하드웨어 | 담당 | LLM decode 기여 |
|----------|------|----------------|
| **통합 메모리 대역폭** | 토큰 생성 속도(decode) | ⭐⭐⭐ 결정적 |
| **통합 메모리 용량** | 모델 적재 가능 여부(게이트) | ⭐⭐ |
| **GPU 코어** | prefill·배칭·이미지생성·VLM·학습 | ⭐ (decode 무관) |
| **NPU(ANE)** | 저전력 작은 모델·시스템 ML | ✕ (현재 거의 미사용) |
| **CPU** | 토크나이저·스케줄링·일부 레이어(offload) | △ |

---

## 6. Mac 선택 기준 (용도별)

```
LLM 채팅·생성 위주 (단일 사용자):
  1순위 대역폭(칩 등급 Max↑, Ultra 유리) → 2순위 용량 → 코어·NPU는 부차
  예) decode 속도는 M_ Pro보다 Max, Max보다 Ultra (대역폭 차이)

이미지 생성·영상·RAG 긴 컨텍스트 병행:
  GPU 코어 많은 상위 빈도 의미 있음 (compute-bound 작업)

다수 사용자 서빙:
  Mac은 부적합 — vLLM(PagedAttention·Continuous Batching)이 CUDA 전용,
  멀티노드 대역폭 합산 불가 → NVIDIA로 ([[../../ai/GPU-Inference-Decision-Guide]])

NPU:
  LLM 관점에선 구매 결정 요소 아님 (시스템이 알아서 사용)
```

---

## 7. 한 줄 결론

> **Mac에서 LLM 채팅 속도(decode)는 GPU 코어도 NPU도 아닌 "통합 메모리 대역폭"이 결정한다.** GPU 코어는 prefill·이미지생성·배칭 같은 compute-bound 작업에, NPU는 저전력 작은 모델에 기여할 뿐, decode 천장은 대역폭에 묶여 있다. 추론용 Mac은 코어 수보다 대역폭(칩 등급)과 메모리 용량을 보고 고른다.

---

## 관련
- [[Apple-M5-Max]] — M5 Max 상세 스펙 (이 원리의 구체 사례)
- [[../../ai/LLM-Inference-Bandwidth]] — TPS = 대역폭/모델크기, Prefill vs Decode
- [[../../ai/GPU-Inference-Decision-Guide]] — 추론 GPU 선택 (NVIDIA 포함)
- [[../../ai/LLM-System-Performance-Analysis]] — capacity 산정, memory-bound 분석
- [[../../ai/Inference-Optimization]] — 양자화·KV cache (대역폭 절감)
