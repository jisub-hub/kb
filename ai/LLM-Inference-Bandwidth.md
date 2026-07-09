---
tags:
  - ai
  - inference
  - hardware
  - gpu
  - bandwidth
  - tps
  - performance
  - deep-dive
created: 2026-06-16
---

# LLM 추론 속도의 물리학 — TPS는 대역폭이다

> [!insight] 핵심 주장
> LLM 토큰 생성(decode) 속도는 **GPU 메모리 대역폭 / 모델 크기**로 거의 결정된다.
> FLOPS(연산 성능)가 아니다. 이건 추측이 아니라 측정 가능한 물리적 사실이다.

---

## 1. 왜 대역폭인가 — 산술 강도 분석

LLM이 토큰 하나를 생성할 때 일어나는 일:

```
[토큰 1개 생성 = 모델 가중치 전체를 VRAM에서 읽기]

70B 모델 (fp16):
  파라미터 수:  70,000,000,000
  바이트/파라미터: 2 (fp16)
  가중치 총 크기: 140GB

  → 토큰 1개 생성 = 140GB를 VRAM에서 읽음

H100 SXM5 메모리 대역폭: 3,350 GB/s
  → 이론적 TPS = 3,350 / 140 ≈ 24 tokens/s

RTX 4090 메모리 대역폭: 1,008 GB/s
  → 이론적 TPS = 1,008 / 140 ≈ 7 tokens/s

실측치와 놀랍도록 일치한다.
```

이를 컴퓨터 아키텍처 용어로 **산술 강도(Arithmetic Intensity)** 라 한다:

```
산술 강도 = 연산량(FLOPs) / 메모리 전송량(Bytes)

행렬-벡터 곱 (토큰 1개 생성):
  행렬 크기: N×M
  연산: 2×N×M FLOPs
  메모리 읽기: 2×N×M bytes (fp16 가중치)
  산술 강도 ≈ 1 FLOP/byte

H100의 compute-to-bandwidth 비율: ~560 FLOP/byte
  → GPU 연산 능력의 1/560만 사용됨
  → 나머지 559/560은 메모리 기다리며 놀고 있음
  → 완전한 메모리 대역폭 병목
```

**GPU의 수천 TFLOPS가 있어도, 대역폭이 막히면 그 연산 능력은 의미없다.**

---

## 2. Prefill vs Decode — 두 단계는 병목이 다르다

LLM 추론에는 물리적으로 다른 두 단계가 있다.

```
[Prefill 단계: 프롬프트 처리]
  입력: 사용자 질문 + RAG 컨텍스트 (예: 2000 토큰)
  연산: 2000개 토큰을 한꺼번에 처리
  수학: 행렬 × 행렬 (Matrix-Matrix Multiply)
  산술 강도: 높음 (batch_size 만큼 선형 증가)
  병목: FLOPS (연산 능력)
  특성: 긴 컨텍스트일수록 선형으로 느려짐

[Decode 단계: 토큰 생성]
  입력: 이전 토큰 1개
  연산: 토큰 1개씩 순차 생성
  수학: 행렬 × 벡터 (Matrix-Vector Multiply)
  산술 강도: 극히 낮음 (~1 FLOP/byte)
  병목: 메모리 대역폭
  특성: 시퀀스 길이와 거의 무관, GPU BW에만 의존
```

```
실제 레이턴시 예시 (Claude Sonnet 급, 2000토큰 컨텍스트, 500토큰 출력):

Prefill: 2000 토큰 × (모델 처리) ≈ 300~800ms     ← FLOPS 병목
Decode:  500 토큰 × (1/TPS) ≈ 500/24 ≈ 20초      ← 대역폭 병목

→ 출력이 길수록 Decode가 지배적
→ RAG로 컨텍스트가 길어지면 Prefill도 느려짐 (두 단계 모두 타격)
```

---

## 3. 하드웨어별 실측 비교

```
[70B 모델 fp16, 단일 배치 decode TPS]

GPU             메모리 대역폭  이론 TPS   실측 TPS
──────────────────────────────────────────────────
RTX 3090        936 GB/s       6.7        ~5~6
RTX 4090        1,008 GB/s     7.2        ~6~7
RTX 5090        1,790 GB/s     12.8       ~10~12   (추정)
A100 80GB       2,000 GB/s     14.3       ~12~14
H100 SXM5       3,350 GB/s     23.9       ~20~24
H200            4,800 GB/s     34.3       ~30~34
B200            8,000 GB/s     57.1       ~50~55   (추정)

[7B 모델 fp16]
RTX 4090        1,008 GB/s     72         ~60~70
RTX 3090        936 GB/s       67         ~55~65
Apple M2 Max    400 GB/s       28         ~25~30
──────────────────────────────────────────────────
TPS ≈ 대역폭 / 모델크기  공식이 실측과 90% 이상 일치
```

**"그래픽카드가 달라도 동일한 결과를 내놓는다"는 관찰:**
- 출력(토큰)은 동일: 같은 가중치, 같은 알고리즘 → 결정론적
- 속도만 다름: 순전히 대역폭 차이
- CUDA 코어 수, Tensor Core 수는 decode TPS에 거의 영향 없음

---

## 4. 배칭이 방정식을 바꾼다

단일 요청에서 배치 처리로 가면 산술 강도가 변한다:

```
배치 크기 B일 때:
  연산: 2×N×M×B FLOPs
  메모리 읽기: 2×N×M bytes (가중치는 한 번만 읽음)
  산술 강도: B FLOP/byte

H100의 compute-bandwidth 비율 ~560일 때:
  B < 560  → 대역폭 병목 (메모리 병목)
  B > 560  → 연산 병목 (FLOPS가 한계)
  B = 560  → Roofline의 정점 (최대 효율)
```

```python
# 직관적 이해
# 70B 모델, 140GB 가중치, H100 3.35TB/s 대역폭

# 배치 1 (단일 사용자):
#   가중치 140GB 읽기 → 토큰 1개 생성
#   시간: 140GB / 3350 GB/s = 0.042초
#   = 24 tokens/s (1명)

# 배치 100 (100명 동시):
#   가중치 140GB 읽기 → 토큰 100개 생성
#   시간: 140GB / 3350 GB/s = 0.042초  ← 같음!
#   = 2,400 tokens/s (100명 합산)
#   = 24 tokens/s per user  ← 1인당 같음

# 핵심: 배칭이 처리량(throughput)을 높이지
#       개인 레이턴시(latency)를 낮추지는 않는다
```

vLLM, TGI 같은 추론 서버가 하는 일의 핵심이 이것이다:
**가중치를 1번 읽는 동안 최대한 많은 요청을 처리해서 처리량(TPS 합산)을 극대화.**

---

## 5. 양자화 — 대역폭 문제의 가장 직접적 해결

```
[양자화가 TPS에 미치는 영향]

           모델 크기   70B 기준   H100 이론 TPS
──────────────────────────────────────────────
fp32       4 bytes    280GB       12
fp16       2 bytes    140GB       24
int8       1 byte      70GB       48
int4       0.5 bytes   35GB       96     ← GGUF Q4, AWQ, GPTQ
int2       0.25 bytes  17.5GB    192     ← 품질 심각하게 저하

→ 양자화는 VRAM을 줄이는 게 아니라
  '토큰당 읽어야 할 바이트 수'를 줄이는 것
→ 대역폭 병목을 직접 완화
→ TPS가 양자화 비율만큼 선형으로 향상
```

```
실용적 결론:
  RTX 4090 (1TB/s) + 70B int4 (35GB)
  TPS = 1008 / 35 ≈ 28 tokens/s

  H100 (3.35TB/s) + 70B fp16 (140GB)
  TPS = 3350 / 140 ≈ 24 tokens/s

  → 소비자용 4090 + int4 양자화 ≈ H100 fp16
  → 물론 정확도는 fp16이 높지만, 속도는 비슷해질 수 있다
```

---

## 6. 멀티 GPU — 텐서 병렬화로 대역폭 합산

```
[텐서 병렬화 (Tensor Parallelism)]

H100 × 1:  대역폭 3.35 TB/s, 모델 70B fp16 140GB
  → TPS ≈ 24

H100 × 2 (텐서 병렬):
  각 GPU가 모델 절반(70GB)을 담당
  두 GPU가 동시에 자기 절반의 가중치를 읽음
  → 실효 대역폭 = 3.35 × 2 = 6.7 TB/s
  → TPS ≈ 48

H100 × 8 (텐서 병렬):
  실효 대역폭 = 3.35 × 8 = 26.8 TB/s
  → TPS ≈ 192
  → 단 GPU 간 통신 오버헤드 발생 (NVLink: 900GB/s 양방향)
```

단, GPU 간 통신 때문에 선형 스케일이 안 된다:
- 4개: ~3.5배 (87.5% 효율)
- 8개: ~6배 (75% 효율)
- NVLink가 PCIe보다 훨씬 낫지만, 여전히 오버헤드 있음

---

## 7. KV Cache — 시퀀스가 길어질수록 추가 대역폭 비용

```
[KV Cache 크기 공식]
  크기 = 2 × 레이어 수 × 헤드 수 × 시퀀스 길이 × 헤드 차원 × 2bytes

Llama 3 70B 예시:
  레이어: 80
  GQA 헤드: 8 (KV)
  헤드 차원: 128
  시퀀스 4096:
    2 × 80 × 8 × 4096 × 128 × 2 = 1,342MB ≈ 1.3GB

  시퀀스 32768 (32K):
    → 10.7GB  ← 가중치 외 추가 VRAM 소비

매 decode step에서:
  - 가중치 읽기: 140GB (고정)
  - KV Cache 읽기: seq_len에 비례
  - 시퀀스가 길수록 decode TPS 하락
```

RAG에서 컨텍스트를 많이 넣을수록 Prefill도 느려지고, KV Cache도 커져서 Decode도 느려진다. **RAG는 두 단계 모두에서 속도를 깎아먹는다.**

---

## 8. CUDA 최적화들이 실제로 하는 일

```
[Flash Attention]
  문제: Attention 계산 중 Q,K,V 행렬을 VRAM에 썼다 읽음
  해결: SRAM(L1 캐시)에서 타일 단위로 처리 → VRAM 왕복 감소
  효과: Prefill에서 메모리 접근 O(n²) → O(n), 속도 2~4배
  대역폭 관점: Attention 계산의 메모리 병목 완화 (decode TPS는 개선 적음)

[Continuous Batching (vLLM)]
  문제: 요청마다 다른 시퀀스 길이 → GPU 활용률 저하
  해결: 실시간으로 새 요청을 진행 중인 배치에 삽입
  효과: 처리량 3~5배 향상 (개인 레이턴시는 개선 없음)

[Paged Attention (vLLM)]
  문제: KV Cache 할당이 비효율적 (최대 시퀀스 길이로 미리 할당)
  해결: 페이지 단위 동적 할당 (OS 가상 메모리와 동일 원리)
  효과: VRAM 사용량 25~50% 절감 → 더 많은 배치 가능

[Speculative Decoding]
  아이디어: 작은 draft 모델로 여러 토큰 예측 → 큰 모델로 한 번에 검증
  조건: 검증 통과율 높아야 함 (도메인 특화 draft 모델에서 효과적)
  효과: 최대 2~3배 TPS 향상 가능
  대역폭 관점: draft 모델(작음) + 검증(큰 모델 1회) → 가중치 읽기 감소

[양자화 (GPTQ/AWQ/GGUF)]
  앞서 설명한 것처럼 직접적 TPS 향상
  AWQ가 GPTQ보다 품질 유지 면에서 우수 (2024 기준)
```

---

## 9. Apple Silicon이 흥미로운 이유

```
[통합 메모리 아키텍처의 의미]

일반 PC:
  CPU RAM (DDR5): 96 GB/s
  GPU VRAM (GDDR7): 1,008 GB/s (RTX 4090)
  CPU↔GPU 전송: PCIe 5.0 × 16 = 64 GB/s   ← 병목

Apple M5 Max:
  통합 메모리 (LPDDR5X): ~600 GB/s
  CPU와 GPU가 같은 메모리 공유
  전송 병목 없음

→ M5 Max로 70B int4 모델 (35GB):
  TPS = 600 / 35 ≈ 17 tokens/s
  (H100 fp16의 70% 수준)

→ 소비 전력: M5 Max ~15W vs H100 700W
  전력당 TPS: M5 Max가 훨씬 효율적

→ 모델이 70GB 넘으면:
  일반 GPU: 다중 GPU 필요 (NVLink 오버헤드)
  M5 Max Ultra: 통합 메모리 최대 192GB → 싱글 디바이스에서 처리
```

---

## 10. Roofline Model — 공식 프레임워크

이 분석 전체를 하나의 그래프로 표현한 것이 Roofline Model이다:

```
TPS (log)
  │
  │  ╱ 대역폭 한계선 (기울기 = 대역폭)
  │ ╱
  │╱___________  연산 한계선 (수평, = peak FLOPS)
  │
  └─────────────────── 산술 강도 (FLOP/byte, log)
     ↑
  decode 위치 (~1 FLOP/byte)
  → 대역폭 한계선에 갇혀 있음

배치 크기 늘릴수록 오른쪽으로 이동
→ 어느 순간 연산 한계선에 닿으면 FLOPS가 병목
```

LLM decode는 항상 Roofline의 왼쪽 하단에 있다. 대역폭 한계선 아래로는 내려갈 수 없다.

---

## 11. 결론 — 사용자 관찰에 대한 직접 답변

```
"TPS는 대역폭 아닌가?"
→ 맞다. 정확히는: TPS_decode ≈ 메모리 대역폭 / 모델 바이트 크기

"모델 크기에 따라 VRAM을 먹고 올라가서 그래픽카드가 달라도 동일한 결과를 내놓지 않는가?"
→ 맞다. 출력(토큰)은 같고, 속도만 다르다.
   (같은 가중치 → 같은 행렬곱 → 같은 결과)

"CUDA 차이로 누가 더 빠르게 할 순 있겠지만"
→ 맞다. CUDA 최적화는 대역폭 이용 효율을 높인다.
   Flash Attention, Paged Attention 등은
   '대역폭을 더 쓴다'가 아니라 '불필요한 대역폭 낭비를 줄인다'

"GPU 성능과 대역폭을 기준으로 속도가 결정되는 것 같다"
→ Decode 단계에서는: 대역폭 >> FLOPS (성능 기여도 비교 불가)
→ Prefill 단계에서는: FLOPS가 중요 (긴 프롬프트 처리 속도)
→ 실용적으로 TPS를 높이려면:
   1순위: 양자화 (모델 크기 줄이기 = 대역폭 요구 직접 감소)
   2순위: 더 많은 대역폭 GPU
   3순위: 멀티 GPU 텐서 병렬 (대역폭 합산)
   4순위: CUDA 최적화 (효율 개선)
   5순위: 배칭 (처리량 향상, 개인 레이턴시는 불변)
```

---

## 12. RAG 맥락으로 다시 연결

```
이전 논의 ("RAG는 느리다")와 연결:

RAG가 컨텍스트를 2000 토큰 추가하면:
  Prefill 시간: +비례하여 증가 (FLOPS 병목)
  KV Cache 크기: +비례하여 증가
  Decode TPS: KV Cache 읽기 오버헤드로 소폭 감소

→ "RAG가 느리다"의 물리적 원인:
   1. Prefill에서 더 많은 토큰을 FLOPS로 처리
   2. KV Cache 증가로 대역폭 요구 증가
   3. 하지만 여전히 총 지연의 70~80%는 Decode
   
→ RAG를 빠르게 하려면:
   컨텍스트를 짧게 (정확한 검색으로 Top-3, 500토큰)
   = 정확한 검색 → Prefill 단축 + KV Cache 감소
   = 여기서도 wiki 구조화의 간접 효과 있음
```

---

## 13. 관련
- [[Inference-Optimization]] — KV Cache, 양자화, vLLM, Speculative Decoding 구현
- [[RAG-Latency-Reality]] — RAG 레이턴시 분해, Semantic Cache
- [[RAG-vs-Knowledge-Systems]] — LLM Wiki vs RAG 구조적 차이
- [[../hardware/gpu-workstation/DGX-Spark]] — H100/B200 스펙, NVLink 대역폭
- [[../hardware/gpu-workstation/Apple-M5-Max]] — 통합 메모리 아키텍처, mlx-lm 추론
