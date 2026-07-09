---
tags:
  - ai
  - gpu
  - inference
  - decision-guide
  - hardware
  - capacity-planning
created: 2026-06-17
---

# GPU 추론 의사결정 가이드
## — "추론만이 목적일 때, 어떤 GPU를 어떻게 고를 것인가"

> 이 문서는 [[LLM-GPU-Complete-Analysis|v2]] · [[LLM-System-Performance-Analysis|v3]]의 이론을 **실전 선택 절차**로 압축한 것이다. 학습이 아닌 **추론 전용** 관점에서, 단일/다수 사용자 × 모델 크기 × 예산에 따라 카드를 한 줄로 못 박는다.

> [!important] 이 문서의 위상 — 구매/구성 전 1차 필터(sizing rule)
> 여기서 나오는 카드 추천과 TPS 수치는 **구매·구성 전 1차 sizing**용이다. 최종 capacity plan은 아니다.
> 실제 SLA 산정은 반드시 **KV cache + prefill budget + η(0.5~0.8) + p95 TTFT/ITL**까지 넣어 [[LLM-System-Performance-Analysis|v3]] 16절 절차로 마무리한다.
> 핵심 한 줄: **한 장 적재 판단은 weight만이 아니라 `weight + 최대 KV + overhead ≤ VRAM × 0.80~0.85`로 한다.**

---

## 0. 단 하나의 원칙

> **학습은 Tensor Core FLOPS에 돈을 쓰고, 추론(decode)은 대역폭과 VRAM에만 돈을 쓴다.**

```
decode TPS ≈ HBM 대역폭 / 모델 크기   (단일 사용자, memory-bound)

→ FLOPS는 decode에서 거의 안 쓰인다 (prefill TTFT에만 영향)
→ 그래서 H100 같은 "학습 괴물"은 추론 전용엔 가성비가 나쁘다
→ 추론용 카드 우선순위: ① VRAM(적재) → ② 대역폭(속도) → ③ FLOPS(부차)
```

---

## 1. 2단계 게이트 — 모든 결정의 출발점

```
게이트 1: 모델이 카드 한 장(VRAM)에 들어가는가?
  YES → 데이터 병렬(DP)로 확장. NVLink 불필요. 컨슈머/프로 카드 OK
  NO  → 텐서 병렬(TP) 강제 → NVLink 필요 → 데이터센터 카드(A100/H100+)

게이트 2: 들어간다면, 대역폭이 충분한 TPS를 주는가?
  TPS ≈ 대역폭 / 모델크기 로 계산 → SLA와 비교
```

**게이트 1이 거의 모든 것을 결정한다.** 한 장에 들어가면 문제가 단순해지고(저렴), 안 들어가면 NVLink·TP·고가 카드로 복잡해진다.

> [!warning] "한 장 적재"는 weight만 보면 안 된다
> 모델 가중치만으로 판단하면 위험하다. KV cache·CUDA context·workspace·fragmentation을 더해야 한다.
> ```
> VRAM_required
>   ≈ 모델파라미터 × bytes × 1.05~1.15   (metadata 포함)
>    + batch × context_len × KV_bytes/token
>    + runtime/workspace 여유
>
> 안전 적재 조건:  VRAM_required ≤ VRAM × 0.80~0.85
> ```
> 즉 "weight가 VRAM보다 작다"가 아니라 **"weight + 최대 KV + 오버헤드가 VRAM의 80~85% 이하"**여야 실서비스 가능하다.

모델 크기 기준 (가중치, INT4 ≈ 파라미터 × 0.5 byte):

| 모델 | FP16 | FP8 | INT4 | 한 장 적재 (실서비스 기준) |
|------|------|-----|------|--------------------------|
| 7~8B | 16GB | 8GB | 4GB | 매우 쉬움 (어떤 카드든) |
| **24B** | 48GB | 24GB | 12GB | 24GB 카드 가능 (context/batch 여유 적음) |
| **32B** | 64GB | 32GB | 16GB | **32GB 권장**, 24GB는 context/batch 제한 조건부 |
| 70B | 140GB | 70GB | 35GB | INT4면 80GB(H100/H200) 또는 96GB(RTX PRO 6000) |
| 405B | 810GB | 405GB | ~203GB | 단일 불가 → 멀티 GPU + TP (실서비스는 NVLink) |

---

## 2. 의사결정 트리

```
[추론 전용 GPU 선택]
│
├─ 사용자 = 나 혼자 (단일/소수)?
│   │
│   ├─ 모델 ≤ 13B
│   │    → RTX 5090 (32GB)         가성비 최강, INT4로 100+ TPS
│   │    → (로컬/저전력) M5 Max
│   │
│   ├─ 모델 24~32B
│   │    → RTX 5090 (32GB)         INT4/FP8 한 장, ~75~150 TPS ★sweet spot
│   │    → 풀정밀/여유: RTX PRO 6000
│   │
│   ├─ 모델 70B
│   │    → RTX PRO 6000 (96GB)     FP8/INT4 한 장, NVLink 회피
│   │    → (로컬) M5 Ultra 256GB+
│   │
│   └─ 모델 100B+ / 405B
│        → 한 장 불가 → TP 필요
│        → 실서비스: H100/H200/B200 ×N + NVLink/NVSwitch (필수)
│        → 실험/저속: PCIe-only 멀티 GPU도 동작은 함 (RTX PRO/5090은 비권장)
│        → (로컬 타협) M5 Ultra 512GB로 양자화 적재 (느림)
│
└─ 사용자 = 다수 (서빙)?
    │
    ├─ 모델이 한 장에 들어감 (≤32B, 또는 70B를 96GB에)
    │    → [데이터 병렬] 카드 ×N + 로드밸런서
    │    → 5090 ×N (≤32B) / RTX PRO 6000 ×N (70B)
    │    → NVLink 불필요, 선형 확장
    │
    └─ 모델이 한 장에 안 들어감 (70B FP16, 100B+)
         → [텐서 병렬] NVLink 필수
         → H200 (추론 특화) > H100, 또는 B200
         → v3 14절 capacity 표로 GPU 수 산정
```

---

## 3. 시나리오별 최종 추천표

| 사용자 | 모델 | 1순위 추천 | 대안 | 핵심 이유 |
|--------|------|-----------|------|----------|
| 단일 | 7~13B | RTX 5090 | M5 Max(로컬) | 한 장, 가성비 |
| 단일 | **24~32B** | **RTX 5090 (32GB)** | RTX PRO 6000 | sweet spot, ~150 roof / ~97 실효 |
| 단일 | 70B | RTX PRO 6000 | M5 Ultra | 96GB 한 장, NVLink 회피 (~33 실효) |
| 단일 | 405B | M5 Ultra 512GB | H100 ×N | 로컬은 용량, 속도는 DC |
| 다수 | ≤32B | 5090 ×N (DP) | RTX PRO 6000 ×N | 처리량 ×N, NVLink 무관 |
| 다수 | 70B | **H200** ×N | RTX PRO 6000 ×N(DP) | 추론 특화 BW/VRAM |
| 다수 | long-context | H200 / B200 | — | KV Cache VRAM 여유 |
| 엣지/현장 | ≤13B | Jetson AGX Thor | M5 | 저전력·오프라인 |

---

## 4. 추론 관점 카드 카탈로그

| 카드 | VRAM | 대역폭 | NVLink | DC 라이선스 | 추론 포지션 |
|------|------|--------|--------|------------|------------|
| RTX 4090 | 24GB | 1.0 TB/s | ❌ | 제약 | 소형 모델 입문 |
| **RTX 5090** | 32GB | 1.79 TB/s | ❌ | 제약 | ≤32B 가성비 최강 |
| **RTX PRO 6000 (Workstation)** | 96GB | 1.79 TB/s | ❌ | ✅ | 70B 한 장, 로컬/워크스테이션 |
| **RTX PRO 6000 (Server)** | 96GB | 1.60 TB/s | ❌ | ✅ | 70B DP 서빙 (대역폭 ↓ 변형) |
| L40S | 48GB | 864 GB/s | ❌ | ✅ | 추론 전용 DC 가성비 |
| A100 80GB | 80GB | 2.0 TB/s | ✅ | ✅ | 범용, TP 가능 |
| H100 SXM | 80GB | 3.35 TB/s | ✅ | ✅ | 고성능(학습 겸용, 추론엔 과투자) |
| **H200 SXM** | 141GB | 4.8 TB/s | ✅ | ✅ | **추론 특화** (70B 다수 사용자) |
| B200 | 192GB | ~8 TB/s | ✅ | ✅ | 차세대, 단일장 70B 여유 |

> [!note] RTX PRO 6000 변형 주의
> **Workstation Edition** = 96GB GDDR7 ECC, **1,792 GB/s**. **Server Edition** = 96GB, **1,597 GB/s** (대역폭 ↓). 둘 다 **NVLink 없음** → 70B 단일 카드/DP replica에는 적합하지만 **405B 같은 대형 TP 서빙엔 부적합**(PCIe 병목). 카드 구매 시 어느 에디션인지 반드시 확인.

> **추론 전용 가성비 라인**: 5090(소형) → RTX PRO 6000(70B 한 장) → H200(70B 다수 사용자 TP). H100은 학습 겸용일 때만 정당화된다.

---

## 5. 양자화 — 게이트 1을 통과시키는 열쇠

```
가중치 양자화 (가장 먼저):
  FP16 → INT4(AWQ/GPTQ): 모델 1/4, TPS 4×, 동시 사용자 4×
  → 70B(140GB→35GB)를 80GB 한 장에 넣는 핵심

KV Cache 양자화 (그 다음):
  FP16 → FP8: KV 절반, 동시 사용자 2× (정확도 <0.3%, 안전)
  → 다수 사용자 capacity의 결정타 (v3 7절)

병용: 가중치 INT4 + KV FP8 → FP16 대비 동시 사용자 ~8×
```

> Apple은 CUDA용 AWQ/GPTQ 대신 GGUF(llama.cpp)·MLX 양자화를 쓴다. 대역폭 이득 원리는 동일(Metal 커널이 on-the-fly dequant).

---

## 6. DP vs TP — 다수 사용자의 갈림길

| | 데이터 병렬 (DP) | 텐서 병렬 (TP) |
|--|------------------|----------------|
| 방식 | 카드마다 모델 전체 복제 | 모델을 카드 간 분할 |
| 얻는 것 | **처리량 ×N** (개별 속도 동일) | 개별 속도 ×N (대역폭 합산) |
| 통신 | 거의 없음 | 매 레이어 All-Reduce |
| NVLink | 불필요 | **사실상 필수** |
| 조건 | 모델이 한 장에 들어감 | 모델이 한 장에 안 들어감 |
| 컨슈머 카드 | ✅ 적합 | ❌ PCIe 병목 (실측 40~60%) |

```
"여러 장 = 대역폭 ×N"은 TP일 때만. DP는 처리량 ×N (개별 TPS는 그대로)
→ 다수 사용자가 원하는 건 보통 "처리량 ×N" = DP
→ 한 장에 들어가는 모델이면 DP가 정석 (NVLink 불필요)
```

---

## 7. 로컬 / 저전력 옵션

| 기기 | VRAM | 대역폭 | 전력 | 포지션 |
|------|------|--------|------|--------|
| M5 Max 128GB | 128GB | ~600 GB/s | ~92W | 단일 사용자 로컬, 조용·저전력 |
| M5 Ultra(추정) | 256~512GB | ~1.1 TB/s | ~150W | 70B 풀정밀/405B 양자화 로컬 |
| DGX Spark | 128GB | 273 GB/s | 170W | CUDA 개발용 데스크탑 |
| Jetson AGX Thor | 128GB | 273 GB/s | 130W | 엣지·로보틱스·현장 |

```
단일 사용자 로컬 추론 → 대역폭 높은 Mac (M5 Max/Ultra)이 DGX Spark보다 빠름
  (DGX Spark 273 GB/s < M5 Max 600 GB/s, 단일 사용자는 대역폭 게임)
CUDA 의존·파인튜닝 겸용 → DGX Spark
현장·임베디드 → Jetson
```

> 단, **다수 사용자 서빙은 Mac이 부적합** — vLLM/PagedAttention/Continuous Batching이 CUDA 전용이고, 멀티노드 대역폭 합산이 안 된다. 다수 사용자 = NVIDIA.

---

## 8. 흔한 함정 (반드시 피하라)

```
❌ "1,979 TFLOPS" 그대로 산정  → Sparse 수치. Dense는 절반(989.5)
❌ 4090/5090 멀티로 70B TP      → NVLink 없어 PCIe All-Reduce 병목
❌ "RAM 크면 VRAM 대체"          → offload는 PCIe-bound, 수십 배 느림
❌ "여러 장 = 무조건 대역폭 ×N"   → DP는 처리량 ×N, 개별 속도는 그대로
❌ 추론 전용에 H100 과투자        → FLOPS 못 씀. H200/RTX PRO 6000이 효율적
❌ KV Cache를 MHA로 과대계산      → GQA는 1/8 (Llama 3, Gemma 등)
❌ roofline 상한을 SLA로 직행     → 이론의 50~75%만 capacity 인정
❌ GeForce 데이터센터 정식 배포    → EULA 제약. 프로/DC 카드 사용
❌ "한 장 적재"를 weight만 판단    → +KV+overhead ≤ VRAM×0.80~0.85로 봐야 안전
❌ RTX PRO 6000 에디션 미확인     → Workstation 1,792 / Server 1,597 GB/s, 둘 다 NVLink 없음
```

---

## 9. 빠른 참조 치트시트

```
모델 크기로 한 장 적재 판단 (INT4 기준):
  ≤32B  → 24~32GB 카드 (5090)
  70B   → 96GB 카드 (RTX PRO 6000) 또는 80GB(H100/H200) INT4
  405B  → 항상 멀티 GPU + TP + NVLink

단일 사용자 TPS ≈ (대역폭 / 모델크기 INT4) × η    ← roofline은 상한, 실효는 ×η(0.5~0.8)
  5090(1.79TB/s) + 24B(12GB)        ≈ 149 roof → ~97 @η0.65
  RTX PRO 6000 WS(1.79TB/s) + 70B(35GB) ≈ 51 roof → ~33 @η0.65
  RTX PRO 6000 Server(1.60TB/s) + 70B   ≈ 46 roof → ~30 @η0.65
  H200(4.8TB/s) + 70B(35GB)         ≈ 137 roof → ~89 @η0.65

다수 사용자:
  한 장에 들어감 → 카드 ×N 데이터 병렬 (NVLink 불필요)
                   ※ DP는 처리량 ×N, 단일 사용자 TPS는 한 카드 성능 그대로
  안 들어감      → H200/H100 ×N 텐서 병렬 (NVLink 필수)
                   ※ PCIe-only TP는 실험/저SLA용, capacity 크게 할인
  capacity 산정  → v3 14절 + η 0.65, prefill budget·p95 SLA 별도 확인

최종 점검 (v3 16절):
  ① VRAM 적재? ② decode roofline? ③ prefill budget? ④ p95 SLA의 몇 %?
```

---

## 10. 결론 — 한 문단 요약

> 추론 전용이면 **VRAM(적재) → 대역폭(속도) → FLOPS(부차)** 순으로 본다. 모델이 한 장에 들어가면(게이트 1 통과) 데이터 병렬로 저렴하게 끝나고(컨슈머/프로 카드, NVLink 불필요), 안 들어가면 텐서 병렬·NVLink·데이터센터 카드로 비싸진다. 그래서 **24B는 5090 한 장, 70B 개인은 RTX PRO 6000, 70B 다수 사용자는 H200**이 각 구간의 정답이다. 학습 겸용이 아니면 H100은 과투자이고, 다수 사용자에 Mac은 부적합하며, "여러 장 = 대역폭 ×N"은 텐서 병렬에만 성립한다.

---

## 관련 노트
- [[LLM-GPU-Complete-Analysis]] — v2: TPS=대역폭/모델크기, GQA, Dense/Sparse
- [[LLM-System-Performance-Analysis]] — v3: capacity planning, DP/TP, NVLink, SLA 산정
- [[LLM-Inference-Bandwidth]] · [[CUDA-Role-in-LLM]] — 이론 배경
- [[Inference-Optimization]] — 양자화·vLLM 구현
- [[SLLM]] — 소형 모델 (엣지/단일 카드)
- [[../hardware/gpu-workstation/_index|GPU Workstation]] · [[../hardware/edge-ai/Jetson]]
