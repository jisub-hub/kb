---
tags:
  - hardware
  - local-ai
  - decision-guide
  - apple-silicon
  - dgx-spark
created: 2026-06-18
---

# 단일 로컬 AI 머신 선택 — M5 Ultra vs DGX Spark

> [!summary] 한 줄 요약
> "한 대만 산다"면 **워크로드가 갈림길**이다 — 100% CUDA 학습/파인튜닝이면 DGX Spark, 추론·앱·에이전트·데일리 드라이버면 M(Mac Studio). 결정은 벤치 수치만이 아니라 **대역폭(체감 속도)·소프트웨어 생태계 성숙도·다목적성**으로 한다. 미출시 칩 스펙은 추정치임을 전제로.

---

## 1. 결론 매트릭스

| 주 용도 | 선택 | 이유 |
|---------|------|------|
| **추론·앱·멀티 에이전트·데일리 드라이버** | **Mac (M Ultra)** | 대역폭(decode), 다목적, 생태계 안정 |
| **100% CUDA 학습/QLoRA 파인튜닝** | **DGX Spark** | 네이티브 CUDA·TensorRT, prefill 연산력 |
| 둘 다 애매하면 (단일 장비) | **Mac** | 마찰 적고 범용 — 아래 근거 |

→ 워크로드 일반 판단은 [[GPU-Inference-Decision-Guide]], Mac 추론 원리는 [[Apple-Silicon-Inference]].

---

## 2. 속도 — 대역폭이 체감을 지배 (decode)

```
체감 속도 = decode TPS = 대역폭 / 모델크기
  DGX Spark: 273 GB/s   → 70B INT4 ~8 TPS   (느림)
  M Ultra:   ~1.1 TB/s+ → 70B INT4 ~31 TPS  (4배)

에이전트도 긴 코드 "생성"(decode)이 많아 → 대역폭 높은 Mac 유리
DGX Spark는 prefill(긴 입력 읽기)만 연산력으로 우위 → 입력≫출력 단계에 한정
```
→ prefill vs decode 분해는 [[LLM-System-Performance-Analysis]] 3.1.

---

## 3. 메모리 — VRAM·컨텍스트 (에이전트 핵심)

```
DGX Spark: 128GB LPDDR5X 고정
M Ultra:   (추정) 256GB+, M3 Ultra는 이미 512GB 실재

멀티 에이전트는 코드·로그·이력 + KV Cache 누적으로 메모리 빠르게 참
→ 큰 통합 메모리 = 70B + 코딩모델(Qwen Coder 32B) 동시 적재 여유
→ [[Multi-Agent-Coding-Team]]
```

> 에이전트 빌딩엔 raw prefill 속도보다 **VRAM 여유·컨텍스트 공간**이 더 자주 병목이 된다.

---

## 4. ⚠️ 소프트웨어 생태계 마찰 (벤치보다 중요한 실전 변수)

```
DGX Spark(GB10, sm121 계열) 신칩 리스크 (early adopter 보고, 시점·개선 여지 있음):
  - "1 PFLOP FP4"의 NVFP4가 stock vLLM/PyTorch에서 깔끔히 안 먹는 사례
  - 실험 브랜치 컴파일 씨름 → 실패 시 BF16 upcast로 fallback (성능·VRAM 손해)
  - headless DGX OS — 디스플레이 없음, SSH/API 전용

Mac(MLX) 성숙:
  - CUDA만큼 학습엔 못 미치나, 추론·로컬 프로토타이핑은 저마찰
  - Ollama/llama.cpp/MLX 즉시 동작
```

> 단일 장비라면 "모델이 당장 깔끔히 도느냐"가 벤치 수치보다 중요하다. **신칩 생태계 마찰은 일상 생산성을 직접 깎는다.** (단 Reddit 일화 기반 — 단정 말고 시간이 개선할 수 있음을 전제)

---

## 5. 다목적성 (단일 장비 제약의 핵심)

```
Mac Studio: 데일리 드라이버 + IDE + Docker(TimescaleDB/Redis/Nginx) +
            RTSP/비디오 파이프라인 + AI 백엔드 = 한 SoC, 무소음
DGX Spark:  AI 전용 headless 박스, 140~240W, SSH/API 대기
```
→ "한 대만"이면 다목적 Mac이 압도적. (전용 추론 서버 여러 대 환경은 별개)

---

## 6. 반대로 DGX Spark가 이기는 경우

```
- 네이티브 CUDA 필수: QLoRA/풀 파인튜닝, TensorRT-LLM, CUDA 전용 라이브러리 연구
- prefill 지배 워크로드: 긴 입력·짧은 출력(문서 읽고 라우팅·판단형)
- NVIDIA 데이터센터 툴체인 학습/실험 환경 구축이 목적
→ 이 경우 생태계 마찰을 감수하고 CUDA 정합성을 택함
```

---

## 7. 미출시 스펙 주의

```
M5 Ultra는 (이 시점) 미출시 추정 — "256GB / 1.2~1.6 TB/s"는 추정치
  현존 M3 Ultra: 512GB / 819 GB/s (실측)
→ 출시 실측 스펙으로 재확인 후 구매 결정. 추정 수치를 확정 근거로 쓰지 말 것.
```

---

## 8. 현실 권장 — 하이브리드

```
단일 머신(Mac) 로컬 70B = 일상 추론·에이전트 기본
  + 복잡·고난도 작업만 클라우드 API(Claude/GPT) 폴백
  → 프라이버시·무제한 + 품질 한계 보완, 장비도 합리적
"완전 동급을 무리하게 로컬화"보다 로컬+API 조합이 비용·품질·속도 균형
```
→ 하이브리드·비용은 [[LLMOps]], 모델 선택은 [[Open-Source-Models]].

---

## 9. 정리

```
워크로드가 갈림길:
  추론·앱·에이전트·데일리  → Mac (대역폭·다목적·저마찰)  ← 단일 장비 대부분
  100% CUDA 학습/파인튜닝  → DGX Spark (CUDA 정합·prefill)
판단 기준: 벤치 수치 ❌ → 대역폭(체감)·생태계 성숙도·다목적성 ⭕
전제: 미출시 스펙은 추정 / 신칩 마찰은 실전 변수 / 결국 로컬+API 하이브리드
```

---

## 관련
- [[Apple-Silicon-Inference]] — Mac 추론 원리(대역폭=속도)
- [[DGX-Spark]] · [[Apple-M5-Max]] — 각 하드웨어 스펙
- [[GPU-Inference-Decision-Guide]] — 추론 GPU 선택 일반
- [[LLM-System-Performance-Analysis]] — prefill/decode·TTFT
- [[Multi-Agent-Coding-Team]] — 에이전트 빌딩 시 VRAM·하드웨어
- [[LLMOps]] · [[Open-Source-Models]] — 하이브리드·모델
