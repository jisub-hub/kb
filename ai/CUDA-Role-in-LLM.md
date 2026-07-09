---
tags:
  - ai
  - cuda
  - gpu
  - inference
  - hardware
  - deep-dive
created: 2026-06-16
---

# CUDA의 역할 — 대역폭이 천장이라면, CUDA는 그 천장에 닿는 사다리

> [!insight] 핵심 답변
> 대역폭이 **이론적 천장**을 결정한다. CUDA는 **실제로 그 천장에 얼마나 근접하느냐**를 결정한다.
> CUDA 없이 H100을 써도 대역폭의 20~30%밖에 못 쓴다.
> CUDA 최적화로 70~90%까지 올린다. 이게 3~4배 실질 성능 차이다.

---

## 1. 이론과 실측의 갭

```
[70B fp16, H100 기준]

이론 TPS = 3,350 GB/s ÷ 140 GB = 24 tokens/s

CUDA 최적화 없음 (단순 PyTorch):  ~6 tokens/s   (25% 효율)
CUDA 최적화 있음 (vLLM + Flash):  ~20 tokens/s  (83% 효율)

차이: 3.3배

→ 하드웨어는 같다. 소프트웨어가 다르다.
```

왜 이 갭이 생기는가. GPU 메모리 계층 구조 때문이다.

```
[GPU 메모리 계층]

레지스터  ~256KB/SM  ~20TB/s  (각 SM당 최고속)
L1/SRAM   ~20MB      ~20TB/s  (H100 기준, VRAM의 6배 속도)
L2 Cache  ~50MB      ~6TB/s
VRAM      ~80GB      ~3.35TB/s  ← 우리가 말하는 대역폭

→ 데이터를 VRAM에서 읽으면 3.35TB/s
→ SRAM에 캐시해두면 20TB/s로 읽음
→ CUDA의 핵심 일: 데이터를 SRAM/레지스터에 최대한 오래 두기
```

---

## 2. Flash Attention — CUDA 최적화의 가장 극적인 예

Attention 계산에서 무슨 일이 생기는가:

```
[Standard Attention (CUDA 최적화 없음)]

입력: Q, K, V 행렬 (seq_len × d_model)

Step 1: S = Q @ Kᵀ                   → 결과를 VRAM에 저장  (4M 토큰²)
Step 2: P = softmax(S)                → VRAM에서 읽어서 계산 → VRAM에 저장
Step 3: O = P @ V                     → VRAM에서 읽어서 계산

VRAM 왕복: seq_len² 크기의 행렬을 3번 읽고 씀
2000 토큰: 2000² × 2bytes × 3 = 24MB × 3 = VRAM 왕복 72MB
           = 72MB ÷ 3.35TB/s = 0.02ms  (대역폭 측면)

[Flash Attention (CUDA 최적화)]

핵심 아이디어: Q, K, V를 타일(tile) 단위로 SRAM에 올려서 처리
              중간 행렬(S, P)을 VRAM에 절대 쓰지 않음
              SRAM에서 계산하고 최종 O만 VRAM에 저장

VRAM 왕복: Q, K, V 읽기 + O 쓰기만 (seq_len에 선형)
           → O(seq_len²) → O(seq_len)  메모리 복잡도
           → 시퀀스 길이 2000: 3~10배 빠름
           → 시퀀스 길이 32000 (long context RAG): 10~50배 빠름
```

**Flash Attention은 대역폭을 늘린 게 아니다. VRAM을 덜 쓰도록 알고리즘을 재설계한 것이다.**

---

## 3. Fused Kernel — VRAM 왕복을 줄이기

```
[Unfused: 각 연산을 별도 커널로]

Input → LayerNorm → VRAM 저장
      → VRAM 읽기 → Linear → VRAM 저장
      → VRAM 읽기 → SiLU   → VRAM 저장
      → VRAM 읽기 → Linear → VRAM 저장

VRAM 왕복: 6번 (읽기 3 + 쓰기 3)

[Fused: 하나의 커널에서 전부 처리]

Input → [LayerNorm → Linear → SiLU → Linear] → VRAM 저장

VRAM 왕복: 2번 (읽기 1 + 쓰기 1)
중간 결과는 레지스터/SRAM에서만 존재
→ 3배 대역폭 절약
```

CUDA는 이런 연산들을 하나의 커널로 묶는 것을 가능하게 한다. CPU에서는 불가능한 패턴이다.

---

## 4. 양자화 커널 — int4로 읽고 레지스터에서 fp16으로 변환

```
[양자화 없는 naive 구현]

VRAM에서 int4 읽기
→ fp16으로 변환해서 VRAM에 저장  ← 이러면 의미없음 (fp16 대역폭 소비)
→ fp16 matmul 실행

[AWQ/GPTQ CUDA 커널]

VRAM에서 int4 읽기 (4배 작음)
→ 레지스터 안에서 즉석 fp16 변환  ← VRAM 쓰기 없음
→ Tensor Core에서 fp16 matmul 실행

효과: int4 대역폭(35GB)만 소비, fp16 대역폭(140GB) 소비 없음
→ 이것이 양자화가 실제로 빠른 이유
   (단순히 모델이 작아서가 아니라, 커널이 영리해서)
```

---

## 5. Tensor Core — Prefill에서만 진짜 힘을 발휘

```
[Tensor Core의 역할]

H100 FP16 성능:
  일반 CUDA Core:  67 TFLOPS
  Tensor Core:    1,979 TFLOPS  ← 30배 차이

Decode 단계 (행렬 × 벡터):
  산술 강도 ≈ 1 FLOP/byte → 대역폭 병목
  Tensor Core 있어봤자 대역폭에 막혀서 큰 의미 없음
  효과: 미미

Prefill 단계 (행렬 × 행렬):
  산술 강도 ≈ batch_size FLOP/byte
  긴 프롬프트(RAG 컨텍스트 2000토큰) → 산술 강도 높아짐
  → Tensor Core 효과 극대화
  → Flash Attention과 시너지 (SRAM에서 Tensor Core 직접 공급)

요약:
  Decode TPS  → Tensor Core 기여 낮음, 대역폭이 결정
  Prefill 속도 → Tensor Core + Flash Attention이 결정
```

---

## 6. CUDA Graph — 반복 오버헤드 제거

```
[Decode 루프: 토큰을 1개씩 생성]

토큰 1: 커널 A 런치 → 커널 B 런치 → 커널 C 런치 → ... (수백 개)
토큰 2: 커널 A 런치 → 커널 B 런치 → 커널 C 런치 → ...
토큰 3: ...

매번 커널 런치: CPU → GPU 명령 전달 ~1ms/token (CPU 오버헤드)
500토큰: 500ms가 순수 오버헤드

[CUDA Graph]

처음 1회: 전체 커널 실행 순서를 그래프로 캡처
이후 매 토큰: "그래프 실행" 명령 1번 = CPU 오버헤드 ~0.05ms/token

500토큰: 25ms 오버헤드 (20배 감소)

→ 빠른 모델일수록 (7B, 13B) 오버헤드 비율이 커서 CUDA Graph 효과 큼
→ 느린 대형 모델 (70B+)은 생성 자체가 느려서 오버헤드 비율 작음
```

---

## 7. 정리 — CUDA는 무엇인가

```
비유: 고속도로

대역폭  = 고속도로 차선 수 (하드웨어가 결정)
CUDA    = 교통 공학 (같은 차선에서 최대한 많은 차를 흘리는 방법)

교통 공학이 하는 일:
  ① 불필요한 U턴 제거 (Fused Kernel: VRAM 왕복 감소)
  ② 고속 우회도로 활용 (Flash Attention: SRAM 활용)
  ③ 빈 공간 없이 차선 채우기 (연속 배칭: GPU 활용률 극대화)
  ④ 짐을 압축해서 적재 (양자화 커널: int4로 읽고 on-the-fly 압축해제)
  ⑤ 신호 대기 시간 최소화 (CUDA Graph: CPU 오버헤드 제거)

→ 차선(대역폭)을 늘릴 수 없다
→ 하지만 같은 차선에서 20% 이용하던 것을 90%로 올릴 수 있다
→ 이게 4.5배 차이다
```

```
[CUDA 최적화별 주요 기여 영역]

최적화              Decode TPS  Prefill 속도  배치 처리량
──────────────────────────────────────────────────────────
Flash Attention        ★☆☆        ★★★          ★★☆
Fused Kernels          ★★☆        ★★☆          ★★☆
양자화 커널            ★★★        ★★★          ★★★
CUDA Graph             ★★☆        ☆☆☆          ★☆☆
Tensor Core            ★☆☆        ★★★          ★★★
Continuous Batching    ☆☆☆        ☆☆☆          ★★★
Paged Attention        ☆☆☆        ☆☆☆          ★★★
──────────────────────────────────────────────────────────
★★★ = 큰 효과  ★★☆ = 중간  ★☆☆ = 작은 효과  ☆☆☆ = 무관
```

---

## 8. 결론 — 세 줄 요약

```
1. 대역폭이 decode TPS의 이론적 천장을 결정한다.  [하드웨어]
2. CUDA는 그 천장에 실제로 얼마나 근접하느냐를 결정한다.  [소프트웨어]
3. Flash Attention, Fused Kernel, 양자화 커널이 가장 중요하고,
   이것들은 대역폭 사용량 자체를 줄이거나 SRAM 우회로를 활용한다.

→ H100이 RTX 4090보다 빠른 이유: 대역폭 3.3배
→ vLLM이 naive PyTorch보다 빠른 이유: CUDA 효율 3~4배
→ 두 가지가 곱해져서 실제 배포 성능 차이가 10배 이상 나는 것
```

---

## 9. 관련
- [[LLM-Inference-Bandwidth]] — TPS = 대역폭/모델크기, Prefill vs Decode 병목
- [[Inference-Optimization]] — vLLM, Flash Attention, 양자화 구현 상세
- [[../hardware/gpu-workstation/DGX-Spark]] — H100/B200 SRAM/VRAM 스펙
- [[../hardware/gpu-workstation/Apple-M5-Max]] — 통합 메모리의 CUDA 없는 대역폭 활용
