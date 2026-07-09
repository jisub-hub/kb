---
tags:
  - ai
  - research
  - report
  - inference
  - hardware
  - cpu
  - pcie
  - capacity-planning
  - tps
  - kv-cache
created: 2026-06-17
---

# LLM 시스템 성능 완전 분석 (v3)
## — GPU · CPU · System RAM · PCIe 통합 아키텍처 보고서

> **판정 기준**: 이 보고서는 단순 개념 설명이 아니라, 실제 LLM 서빙 인프라를 산정할 때 오류가 나지 않도록 **이론 상한, 메모리 적재 가능성, 실서비스 capacity, 통신 오버헤드, prefill/decode 분리, offload 방식별 병목**을 구분한다. [[LLM-GPU-Complete-Analysis|v2]]의 문제의식("TPS는 무엇이 결정하는가", "CUDA는 무슨 역할인가", "동시 사용자 처리량은 어떻게 계산하는가", "학습/추론 GPU 선택 기준")을 계승해 시스템 수준 보고서로 재구성했다.

---

## 0. 핵심 결론

**LLM decode 추론의 1차 병목은 대부분 FLOPS가 아니라 HBM 대역폭이다.** 단일 사용자 decode에서는 모델 가중치를 매 토큰마다 거의 한 번씩 읽어야 하므로, 단순 상한은 `HBM bandwidth / resident model bytes`에 가깝다. 다만 이 문장은 **dense 모델, low/medium batch, GPU-resident weight, decode-only** 조건에서 가장 잘 맞는다.

**다중 사용자 처리량은 배칭으로 증가한다.** 사용자 1명이든 64명이든 decode step마다 모델 가중치는 한 번 읽고 공유할 수 있기 때문이다. 하지만 사용자마다 별도로 누적되는 KV cache가 VRAM과 HBM bandwidth를 소비하므로, 큰 batch에서는 **KV cache가 처리량과 동시 사용자 수의 새로운 병목**이 된다.

**70B FP16 모델은 약 140GB**이므로 80GB GPU 단일장에는 올라가지 않는다. H100 80GB 단일장에서 70B를 실서비스하려면 대체로 **INT4 weight quantization**이 필요하고, FP16/BF16 유지가 필요하면 **TP=2 이상 또는 H100 NVL/H200급 이상** 구성이 필요하다. H200 141GB도 70B FP16 가중치만 간신히 담는 수준이므로, KV cache와 runtime overhead를 고려하면 단일장 실서비스는 부적절하다.

**System RAM offload는 "RAM 용량이 크니 해결된다"가 아니다.** GPU가 host RAM의 데이터를 소비해야 하는 구조라면 병목은 대체로 `min(System_RAM_BW, PCIe_BW)`이며, PCIe 4.0/5.0 x16의 단방향 대역폭은 H100 HBM3의 수십 분의 일이다. DDR5-5600은 모듈당 약 44.8GB/s, H100 SXM은 3.35TB/s HBM bandwidth다.

**SLA 산정은 roofline 상한을 그대로 쓰면 안 된다.** kernel efficiency, dequantization, scheduler, sampling, CUDA Graph 적용 여부, fragmentation, communication, p95 latency budget을 반영해야 하므로 보수적으로 **이론 TPS의 50~75%**를 capacity로 인정하는 것이 안전하다.

---

## 1. 용어와 산정 기준

이 보고서에서 **TPS는 별도 언급이 없으면 decode output tokens/sec**를 의미한다. Prefill 단계의 "처리 토큰/sec"와 decode 단계의 "생성 토큰/sec"는 병목이 다르므로 같은 지표로 섞으면 안 된다. 단위는 계산 편의를 위해 decimal 기준을 사용한다.

| 기호 | 의미 |
|------|------|
| `P` | 모델 파라미터 수 |
| `W` | GPU에 상주하는 모델 가중치 크기 |
| `B_HBM` | GPU HBM bandwidth |
| `N` | decode batch size (동시에 한 step을 생성하는 sequence 수) |
| `S` | 사용자당 현재 context length |
| `K` | KV cache bytes/token/user |
| `η` | effective utilization factor |
| `TPS_roof` | 통신/스케줄링/커널 비효율을 제외한 이론 상한 |
| `TPS_cap` | SLA 산정용 보수적 capacity |

가중치 크기 1차 근사:

```
FP16/BF16 weight size ≈ P × 2 bytes
FP8/INT8 weight size  ≈ P × 1 byte
INT4 weight size      ≈ P × 0.5 byte + scale/zero/metadata

70B FP16/BF16 ≈ 140GB
70B INT8      ≈ 70GB
70B INT4      ≈ 35GB + metadata
```

> INT4의 35GB는 **하한**에 가깝다. 실제 AWQ/GPTQ 체크포인트는 group size, scale/zero 저장 방식, tensor packing, framework layout에 따라 몇 GB 증가할 수 있다.

---

## 2. Roofline 모델: 왜 decode는 대역폭 문제인가

Roofline 모델은 성능을 연산량과 데이터 이동량의 비율로 본다 [Williams et al., CACM 2009].

```
AI = FLOPs / bytes moved
Machine Balance = Peak FLOPS / Peak bandwidth

AI < Machine Balance → memory-bound
AI > Machine Balance → compute-bound
```

H100 SXM 공식 스펙: FP16 Tensor Core 1,979 TFLOPS, HBM bandwidth 3.35TB/s, NVLink 900GB/s, PCIe Gen5 128GB/s. 이 FP16/BF16 Tensor Core 수치에는 **with sparsity** 주석이 붙어 있으므로, dense LLM 기준 peak는 대략 절반인 **989.5 TFLOPS**로 보는 것이 맞다.

```
H100 SXM dense machine balance
≈ 989.5 TFLOPS / 3.35 TB/s
≈ 295 FLOP/byte
```

FP16 dense decode batch=1: 가중치 2 bytes를 읽어 ~2 FLOPs 수행 → AI ≈ 1 FLOP/byte. 이는 H100 dense machine balance 295보다 훨씬 낮다. 그래서 일반적인 dense LLM decode는 대체로 memory-bound다.

> [!warning] "decode는 언제나 memory-bound"는 과한 표현
> 매우 큰 batch, speculative decoding, MoE, long-context attention, dequantization-heavy kernel, 일부 fused attention backend에서는 단순 `BW / model_size`에서 벗어난다.
> **정확한 표현**: 일반적인 dense LLM의 low/medium-batch GPU-resident decode는 대체로 HBM memory-bound다.

---

## 3. Prefill과 Decode는 다른 문제다

| 단계 | 입력 | 주요 연산 | 병목 | 운영 지표 |
|------|------|----------|------|----------|
| **Prefill** | 프롬프트 전체 L tokens | 큰 GEMM, attention | compute-bound 또는 attention-bound | TTFT |
| **Decode** | 직전 token 1개 | GEMV/small GEMM + KV attention | 대체로 memory-bound | ITL, output TPS |

**Prefill**은 입력 prompt 전체를 병렬 처리한다. L이 충분히 크면 행렬-행렬 곱이 커지고 Tensor Core 활용률이 높아져 compute-bound에 가까워진다. FlashAttention 계열은 attention의 메모리 접근을 줄여 긴 sequence에서 중요하다. FlashAttention-2는 병렬화와 work partitioning을 개선해 A100에서 theoretical max FLOPs/s의 **50~73%** 수준까지 접근한다고 보고한다 [Dao, ICLR 2024].

**Decode**는 autoregressive 구조 때문에 한 번에 다음 token 1개를 생성한다. 매 step마다 모델 가중치를 다시 읽는 효과가 크므로 HBM bandwidth가 1차 상한이 된다.

실서비스에서 중요한 점은 **prefill과 decode가 서로 간섭**한다는 것이다. 긴 prompt 요청이 많이 들어오면 신규 요청의 TTFT가 증가하고, 동시에 이미 decode 중인 요청의 ITL도 악화될 수 있다. 따라서 capacity planning은 반드시 두 예산을 분리해야 한다.

```
Decode budget  = required output tokens/sec
Prefill budget = requests/sec × average input tokens
```

### 3.1 사용자 체감 시간 = TTFT + 생성시간

사용자가 결과물을 다 받기까지의 end-to-end 시간은 두 구간의 합이고, **각 구간의 병목이 다르다.**

```
사용자 체감 시간 (end-to-end)
│
├─ TTFT (Time To First Token) ─── prefill ─── compute-bound (FLOPS)
│    = 입력 토큰 수 × 모델크기 × 2 / FLOPS
│    "질문 보냈는데 첫 글자가 안 나오는" 멈춤 구간
│
└─ 생성시간 = 출력 토큰 수 × ITL ── decode ── memory-bound (대역폭)
     ITL(Inter-Token Latency) = 모델크기 / 대역폭
     "글자가 흘러나오는" 구간

총 시간 = TTFT + (출력 토큰 수 × ITL)
```

| 입력 길이 | 지배 구간 | 결정 요인 |
|----------|----------|----------|
| 짧은 질문 (수십 토큰) | 생성시간(decode) | 대역폭 |
| 긴 컨텍스트·RAG (수천~수만 토큰) | **TTFT(prefill)** | **FLOPS** |

> 그래서 **긴 프롬프트·RAG일수록 prefill이 TTFT 병목**이 된다. "decode는 대역폭"은 첫 토큰 *이후* 얘기일 뿐, 첫 토큰까지는 prefill=FLOPS가 좌우한다.
> 대응: prefix caching(공통 system prompt·RAG 고정부 KV 재사용 → prefill 스킵), chunked prefill(15절), 컨텍스트 축소. FLOPS 높은 GPU는 TTFT에 유리.

---

## 4. 하드웨어 스펙: Dense와 Sparse를 분리하라

인프라 산정에는 "마케팅용 sparse TFLOPS"가 아니라 **dense TFLOPS, HBM bandwidth, VRAM, interconnect**를 따로 봐야 한다.

| GPU | FP16 Dense | Sparse 표기 | HBM BW | VRAM | 해석 |
|-----|-----------|------------|--------|------|------|
| RTX 4090 | ML 서버 주력 아님 | sparse 표기 혼재 | ~1.0 TB/s | 24 GB | 개인 연구/소형 모델 |
| A100 80GB | 312 TFLOPS | 624 TFLOPS | ~2.0 TB/s | 80 GB | 학습/추론 범용 |
| **H100 SXM** | **989.5 TFLOPS** | 1,979 TFLOPS | 3.35 TB/s | 80 GB | 고성능 학습/추론 |
| H200 SXM | 989.5 TFLOPS | 1,979 TFLOPS | 4.8 TB/s | 141 GB | 추론/long-context 유리 |
| B200 HGX | 2.25 PFLOPS | 4.5 PFLOPS | 7.7~8.0 TB/s | 180 GB | 차세대 학습/추론 |

> B200 HGX 180GB 사양은 Lenovo 제품 가이드 기준 FP16 Tensor Core 2.25/4.5 PFLOPS, 180GB HBM3e, 7.7TB/s, NVLink 900GB/s, PCIe Gen5 128GB/s. 2.25/4.5 표기는 "without/with structural sparsity"다.

70B FP16 이론 decode 상한 = `HBM_BW / 140GB`:

| GPU | HBM BW | 70B FP16 이론 상한 | 단일 GPU 적재성 |
|-----|--------|-------------------|----------------|
| RTX 4090 | ~1.0 TB/s | ~7 tok/s | 불가능 |
| A100 80GB | ~2.0 TB/s | ~14 tok/s | 불가능 |
| H100 80GB | 3.35 TB/s | ~24 tok/s | 불가능 |
| H200 141GB | 4.8 TB/s | ~34 tok/s | 가중치만 간신히 |
| B200 180GB | 7.7~8.0 TB/s | ~55~57 tok/s | 가능, KV 여유 있음 |

> 이 표는 **적재 가능성과 대역폭 상한을 분리**해서 읽어야 한다. RTX 4090, A100, H100 단일 GPU의 70B FP16 TPS는 계산상 의미는 있지만 실제로는 모델이 VRAM에 올라가지 않는다.

---

## 5. KV Cache: 동시 사용자 수를 결정하는 핵심

Decode는 이전 token들의 key/value를 계속 참조한다. KV cache 크기는 모델 전체 hidden size가 아니라 **KV head 수**에 의해 결정된다 [Ainslie et al., EMNLP 2023].

```
KV_bytes_per_token = 2 × num_layers × num_kv_heads × head_dim × bytes_per_element
```

Llama 3 70B 대표 config: hidden size 8192, attention heads 64, layers 80, **key-value heads 8**, max position 8192. Meta Llama 3 model card도 8B/70B 모두 GQA를 사용한다고 설명한다.

```
Llama 3 70B GQA, FP16 KV:
  = 2 × 80 × 8 × 128 × 2 bytes
  = 327,680 bytes ≈ 0.328 MB/token

context 2048, 사용자 1명:
  0.328 MB × 2048 ≈ 671 MB ≈ 0.67 GB/user
```

| 구조 | KV heads | FP16 KV/token | context 2048/user |
|------|---------|--------------|------------------|
| MHA 70B 예시 | 64 | ~2.62 MB | ~5.37 GB |
| **Llama 3 70B GQA** | 8 | ~0.328 MB | ~0.67 GB |

> **GQA는 KV cache 요구량을 1/8로 줄인다.** 같은 VRAM에서 수용 가능한 동시 사용자 수가 크게 증가한다.

---

## 6. 다중 사용자 Decode Throughput 공식

정확한 출발점은 step time이다.

```
t_step ≈ (W + N × S × K) / (η_bw × B_HBM)
       + t_launch + t_sampling + t_scheduler + t_comm

Total_TPS ≈ N / t_step
```

순수 roofline 상한 (overhead 제거, η_bw = 1):

```
Total_TPS_roof ≈ N × B_HBM / (W + N × S × K)
```

**작은 batch** (`W >> N × S × K`):
```
Total_TPS ≈ N × B_HBM / W   →  batch가 커질수록 거의 선형 증가
```

**큰 batch 또는 긴 context** (`N × S × K >> W`):
```
Total_TPS ≈ B_HBM / (S × K)   →  총 TPS 포화, 사용자당 TPS는 1/N에 근접
```

---

## 7. H100 80GB, 70B INT4, Llama 3 GQA 예시

가정: H100 SXM 80GB, HBM 3,350GB/s, 70B INT4 W≈35GB, Llama 3 GQA FP16 KV K≈0.328MB/token, S=2048 (사용자당 KV ~0.67GB)

| Batch N | 모델 W | KV 누적 | 총 읽기량/step | 이론 총 TPS | 사용자당 TPS |
|---------|--------|---------|---------------|------------|-------------|
| 1 | 35 GB | 0.67 GB | 35.67 GB | 94 | 94 |
| 8 | 35 GB | 5.37 GB | 40.37 GB | 664 | 83 |
| 32 | 35 GB | 21.47 GB | 56.47 GB | 1,899 | 59 |
| 64 | 35 GB | 42.95 GB | 77.95 GB | 2,751 | 43 |

VRAM 여유 (이상 조건): `80 - 35 = 45GB` → `45 / 0.67 ≈ 67 users`. 하지만 실제로는 CUDA context, allocator reserve, graph capture buffer, activation/workspace, fragmentation, framework overhead가 있으므로 **FP16 KV 기준 H100 80GB 단일장에서는 50명대부터 검증**하는 것이 안전하다.

FP8 KV cache 사용 시 KV 메모리·bandwidth를 대략 절반으로 줄일 수 있다. vLLM 문서는 FP8 KV cache가 cache footprint를 줄여 저장 token 수를 늘리고 throughput을 개선한다고 설명하며, calibration dataset 기반 scale 산정을 권장한다.

| KV dtype | 사용자당 KV @2048 | H100 80GB 이론 사용자 수 | 배포 판단 |
|----------|------------------|------------------------|----------|
| FP16/BF16 | ~0.67 GB | ~67명 minus overhead | 기준 |
| FP8 | ~0.34 GB | ~134명 minus overhead | 실용적, 검증 필수 |
| INT4/FP4 | ~0.17 GB + metadata | 이론상 더 큼 | backend/model별 검증 |

> FP8 KV cache는 "무조건 품질 손실 없음"이 아니다. 모델 구조, attention backend, calibration, long-context 분포에 따라 regression test가 필요하다.

---

## 8. System RAM, PCIe, Offload 병목

System RAM은 용량은 크지만 HBM보다 bandwidth가 훨씬 낮다. DDR5-5600은 모듈당 ~44.8GB/s, dual-channel 데스크톱/워크스테이션은 ~90GB/s급, 고채널 서버는 수백 GB/s급이 가능하다. GPU가 host RAM 데이터를 매 step 소비해야 하면 PCIe가 또 다른 상한이 된다. PCIe 5.0은 32GT/s 규격, PCIe 4.0 x16은 단방향 ~32GB/s급으로 자주 계산된다.

### 8.1 GPU-side offload

host RAM에 있는 weight/KV를 GPU가 가져와 연산하는 구조:

```
t_step_offload
≈ W_HBM / B_HBM
 + W_offload / min(System_RAM_BW, PCIe_H2D_BW)
 + KV_HBM / B_HBM
 + KV_offload / min(System_RAM_BW, PCIe_H2D_BW)
```

극단 예: 70B FP16 140GB를 매 token PCIe 5.0 x16 단방향 64GB/s로 가져온다면:

```
64GB/s / 140GB ≈ 0.46 tokens/s         ← offload
3,350GB/s / 140GB ≈ 23.9 tokens/s      ← HBM resident
```

> **offload는 "느려진다"가 아니라, 구조에 따라 수십 배 느려질 수 있다.**

### 8.2 CPU-side execution

llama.cpp식 일부 layer CPU 실행은 다르다. CPU에 남긴 layer는 GPU로 weight를 매번 복사하는 게 아니라 CPU가 직접 계산한다. 이 경우 병목은 PCIe보다 **CPU roofline**이다.

```
t_layer_cpu ≈ max(
    FLOPs_layer / effective_CPU_FLOPS,
    Bytes_layer / effective_System_RAM_BW
  )
```

AVX/AVX-512/AMX 지원, NUMA 배치, memory channel 수, quantized kernel 품질, thread pinning이 중요하다. PCIe는 주로 CPU layer와 GPU layer 사이 activation 이동에 관여한다.

### 8.3 System RAM 용량 rule of thumb

| 사용 방식 | RAM 산정 |
|----------|---------|
| No offload inference | checkpoint size의 1.5~2배 + OS/runtime 여유 |
| Multi-GPU inference | 총 VRAM의 1~2배를 운영 여유/pinned staging으로 확보 (흔한 구성) |
| GPU-side offload | offloaded weights + offloaded KV + staging + page cache |
| CPU optimizer/offload training | parameters + gradients + optimizer states + dataloader/cache 별도 |

> "총 GPU VRAM × 2"는 수학 공식이 아니라 운영 rule이다. 실제 최소치는 checkpoint format, 로딩 방식, pinned memory 정책, offload 여부에 따라 달라진다.

---

## 9. CPU의 역할

추론 서버에서 CPU는 단순 보조 장치가 아니다. 작은 모델이나 naive PyTorch decode에서는 kernel launch overhead와 Python/runtime control thread가 병목이 될 수 있다. CUDA Graph는 반복 decode graph를 캡처해 CPU launch overhead를 크게 줄인다.

프로덕션 serving에서 CPU가 담당하는 것:

- request admission control
- continuous batching scheduler
- tokenizer / detokenizer
- sampling, logprobs, grammar/structured output
- HTTP/gRPC networking
- KV block allocator
- metrics, tracing, backpressure

> "추론은 단일 코어 클럭만 중요하다"도 틀리고, "코어 수만 많으면 된다"도 틀리다. **지연 민감 serving에서는 단일 코어 성능이, high-throughput serving에서는 scheduler/tokenizer/networking을 처리할 코어 수와 메모리 대역폭이 중요하다.**

학습에서는 CPU 역할이 다르다. DataLoader, tokenization, decompression, augmentation, shuffling, streaming input pipeline이 병목이 될 수 있다. 이미 tokenized dataset과 GPUDirect Storage를 쓰면 CPU 요구량이 줄고, online tokenization이나 복잡한 preprocessing을 하면 GPU 1장당 8~16 CPU core 이상도 필요할 수 있다.

---

## 10. CUDA와 Serving Engine 최적화

CUDA 최적화는 이론 상한을 바꾸기보다 **이론 상한에 가까워지게** 하는 작업이다. 단, quantization과 speculative decoding은 방정식 자체를 바꾼다.

| 최적화 | 주효과 | Decode | Prefill | 배치 처리량 | 주의점 |
|--------|--------|--------|---------|-----------|--------|
| FlashAttention | attention HBM 왕복 감소 | 낮음~중간 | 높음 | 중간 | long-context에서 중요 |
| Fused Kernel | HBM round-trip 감소 | 중간 | 중간 | 중간 | kernel/backend 의존 |
| AWQ/GPTQ INT4 | weight bytes 감소 | 매우 높음 | 중간~높음 | 높음 | 품질/metadata/커널 검증 |
| CUDA Graph | CPU launch overhead 감소 | 중간 | 낮음 | 낮음~중간 | dynamic shape 충돌 가능 |
| Continuous Batching | GPU idle time 감소 | 간접 | 간접 | 매우 높음 | latency scheduler 필요 |
| PagedAttention | KV fragmentation 감소 | 간접 | 간접 | 매우 높음 | block size, eviction 정책 |
| FP8 KV Cache | KV memory/bandwidth 감소 | 중간~높음 | 낮음 | 높음 | calibration/regression 필수 |
| Speculative Decoding | target pass당 복수 token | 높음 | 낮음 | 중간 | acceptance rate 의존 |

Orca는 request 단위가 아니라 **iteration 단위 scheduling과 selective batching**을 제안했고, FasterTransformer 대비 동일 latency 수준에서 최대 **36.9× throughput** 개선을 보고했다 [Yu et al., OSDI 2022].

PagedAttention/vLLM은 KV cache를 block/page 단위로 관리해 낭비를 줄인다. vLLM 논문은 KV cache memory waste를 near-zero로 줄이고 throughput을 **2~4배** 개선한다고 보고한다 [Kwon et al., SOSP 2023].

---

## 11. Dense 모델과 MoE 모델은 다르게 계산해야 한다

```
Dense TPS_roof ≈ B_HBM / total_weight_bytes

MoE VRAM requirement ≈ total parameters       ← 전체 expert 적재
MoE decode bandwidth ≈ active parameters + router/dispatch overhead
```

Mistral AI는 Mixtral 8x7B가 총 **46.7B** parameters이지만 token당 **12.9B** parameters만 사용한다고 설명한다. 따라서 Mixtral 같은 MoE는 같은 총 parameter 수의 dense 모델보다 decode cost가 훨씬 낮을 수 있다.

> 단, MoE 실측 성능은 expert routing, load imbalance, expert parallel communication, batch 내 expert 분포, kernel packing 효율에 영향을 받는다. "active parameter만 읽는다"는 이상식은 **상한**이고, 운영 성능은 더 낮다.

---

## 12. Data Parallel vs Tensor Parallel

### 12.1 Data Parallel (DP)

각 GPU가 모델 전체 복사본을 들고 요청을 나눠 받는다.

**장점**: GPU 간 All-Reduce 없음, 장애 격리 쉬움, serving replica scaling과 잘 맞음, NVLink 없이도 가능.
**단점**: 모델이 각 GPU에 반복 적재되어 VRAM 효율이 낮음.

70B INT4처럼 단일 H100에 들어가는 모델은 DP가 기본 후보다.

```
Total_TPS_DP ≈ Σ [ N_i × B_HBM / (W + N_i × S × K) ]
균등 분배라면 N_i = total_users / num_gpus
```

### 12.2 Tensor Parallel (TP)

모델 weight를 GPU 간 sharding한다. aggregate HBM bandwidth를 사용할 수 있지만 각 layer마다 GPU 간 통신이 발생한다.

```
Total_TPS_TP_roof ≈ N × (G × B_HBM) / (W + N × S × K)   ← G = TP group size

t_step_TP ≈ (W + N × S × K) / (G × B_HBM)
          + t_allreduce + t_launch + t_scheduler
```

H100 SXM은 NVLink 900GB/s와 PCIe Gen5 128GB/s aggregate를 제공한다. **TP에서 선형 확장을 원하면 NVLink/NVSwitch가 사실상 필요하다.** ([[Kubernetes|인프라]] 관점은 별도.)

---

## 13. TP All-Reduce 오버헤드 계산

Llama 3 70B-class 예시 (layers 80, hidden 8192, batch 100, activation FP16 2 bytes, TP=2):

```
AllReduce bytes/step
≈ 2 × layers × hidden_size × batch × activation_bytes
≈ 2 × 80 × 8192 × 100 × 2
≈ 262 MB

PCIe 5.0 x16 단방향 64GB/s:  262MB / 64GB/s  ≈ 4.1 ms
NVLink 900GB/s:              262MB / 900GB/s ≈ 0.29 ms
```

H100 2장 TP=2, batch=100 FP16 KV step bytes:

```
W + N × S × K ≈ 35GB + 100 × 0.67GB ≈ 102GB
aggregate HBM = 2 × 3350 = 6700 GB/s
compute/memory step ≈ 102GB / 6700GB/s ≈ 15.2 ms
```

| 조건 | 통신 시간 | step 대비 |
|------|----------|----------|
| PCIe 5.0 x16 | ~4.1 ms | ~27% |
| NVLink 900GB/s | ~0.29 ms | ~1.9% |

> FP8 KV를 쓰면 memory step은 짧아지지만 All-Reduce traffic은 activation shape에 묶이므로 거의 줄지 않는다. 즉 **FP8 KV + TP 구성에서는 PCIe 통신 비율이 더 커진다.**
>
> **결론: TP=2 이상으로 serving capacity를 뽑아야 한다면 NVLink/NVSwitch를 기본 전제로 둔다.** PCIe-only TP는 동작은 가능하지만 SLA capacity 산정에서는 크게 할인해야 한다.

---

## 14. 실전 Capacity Planning 예시

**목표**: 100명 동시 접속, 사용자당 50 TPS, 70B INT4

```
필요 총 decode TPS: 5,000 tokens/s
모델: 70B INT4 ≈ 35GB,  Context: 2048
KV: Llama 3 70B GQA (FP16 0.67GB/user, FP8 0.34GB/user)
SLA 산정 계수: 이론 TPS × 65%   (decode-only 기준, prefill 별도)
```

### 14.1 H100 구성 비교

H100 `B_HBM = 3,350 GB/s`. 아래 "계산식"은 6/12절 공식을 그대로 적용한 것이다.

| 옵션 | 구성 | 계산식 (이론 TPS) | 이론 TPS | 65% SLA | 판정 |
|------|------|------------------|---------|---------|------|
| A | 2× H100 DP + FP16 KV | `2 × (50×3350 / (35 + 50×0.67))` = 2 × 2,445 | 4,891 | 3,179 | 실패 |
| B | 2× H100 TP=2 + NVLink + FP16 KV | `100×6700 / (35 + 100×0.67)` = 670,000/102 | 6,569 | 4,270 | 실패 |
| C | 2× H100 DP + FP8 KV | `2 × (50×3350 / (35 + 50×0.335))` = 2 × 3,237 | 6,473 | 4,208 | 실패 |
| **D** | **2× H100 TP=2 + NVLink + FP8 KV** | `100×6700 / (35 + 100×0.335)` = 670,000/68.5 | 9,781 | 6,358 | **통과 후보** |
| E | 4× H100 DP + FP8 KV | `4 × (25×3350 / (35 + 25×0.335))` = 4 × 1,931 | 7,723 | 5,020 | 통과, 여유 작음 |
| F | 8× H100 DP + FP16 KV | `8 × (12.5×3350 / (35 + 12.5×0.67))` = 8 × 965 | 7,723 | 5,020 | 통과, 비용 비효율 |

> **계산 메모**
> - DP: 사용자 100명을 GPU 수로 나눠(`N_i = 100/G`) GPU별로 계산 후 합산. TP: aggregate HBM(`G × 3,350`)에 N=100 전체 적용 (통신 overhead 제외한 roofline).
> - E와 F가 동일 값인 이유: GPU당 KV 누적이 둘 다 8.375 GB(`분모 43.375`)로 같고 aggregate 사용자 수가 100으로 같기 때문. (E: FP8·25명/GPU, F: FP16·12.5명/GPU)
> - B/D의 TP roofline은 NVLink 기준. 13절대로 PCIe-only면 ~27% 통신 손실을 추가로 할인해야 한다.

> **2× H100만으로 5,000 TPS SLA를 노리려면 TP=2 + NVLink + FP8 KV가 필요하다.** FP8 KV를 허용하지 않으면 2× H100은 65% SLA 기준으로 부족하다. DP-only 단순성을 중시하면 4× H100 DP + FP8 KV가 최소권, FP16 KV 유지 시 8× H100 수준까지 필요하다.

### 14.2 H200/B200 관점

H200 `B_HBM = 4,800 GB/s`, B200 `B_HBM = 7,700~8,000 GB/s` (dense 기준 범위).

| 옵션 | 계산식 (이론 TPS) | 이론 TPS | 65% SLA | 판정 |
|------|------------------|---------|---------|------|
| 1× H200 + FP16 KV | `100×4800 / (35 + 100×0.67)` = 480,000/102 | 4,706 | 3,059 | 실패 |
| 1× H200 + FP8 KV | `100×4800 / (35 + 100×0.335)` = 480,000/68.5 | 7,007 | 4,555 | 실패 |
| 2× H200 DP + FP8 KV | `2 × (50×4800 / (35 + 50×0.335))` = 2 × 4,638 | 9,275 | 6,029 | 통과 |
| 1× B200 + FP16 KV | `100×(7700~8000) / 102` | 7,549~7,843 | 4,907~5,098 | 경계선 |
| 1× B200 + FP8 KV | `100×(7700~8000) / 68.5` | 11,241~11,679 | 7,307~7,591 | 통과 |

> B200 단일장 + FP16 KV는 7.7TB/s와 8.0TB/s 중 어떤 bandwidth 기준이냐에 따라 65% SLA가 경계선(4,907~5,098)에 걸린다. 실서비스라면 단일 B200 + FP8 KV 또는 다중 replica를 권장한다.
>
> 검증: 위 모든 셀은 `Total_TPS_roof = N × B_HBM / (W + N × S × K)`와 `TPS_cap = TPS_roof × 0.65`에 일관되게 일치한다 (반올림 ±1).

---

## 15. Prefill Budget을 포함한 배포 산정법

7~14절의 decode 산정은 output token 생성 단계만 본다. 실제 서비스에는 request arrival이 있고, 각 request마다 prompt prefill이 발생한다.

```
Prefill FLOPs/sec
≈ requests/sec × input_tokens × 2 × model_params + attention_quadratic_term

예: 70B, 10 req/s, 평균 input 2048 tokens (weight GEMM 항만):
   10 × 2048 × 2 × 70B ≈ 2.87 PFLOP/s
```

H100 dense FP16/BF16 peak ~989.5TFLOPS를 고려하면, prefill이 decode와 같은 GPU pool을 공유할 때 TTFT와 ITL이 동시에 악화될 수 있다. (1,979TFLOPS는 sparsity 수치이므로 dense 산정엔 절반.)

실전 설계 선택지:

1. **Prefill+Decode 같은 pool** — 단순하지만 긴 prompt가 decode ITL 방해
2. **Chunked Prefill** — 긴 prompt를 chunk로 쪼개 decode와 interleave, TTFT/ITL 균형
3. **Prefill/Decode 분리 아키텍처** — compute-heavy prefill GPU와 memory-heavy decode GPU 분리, 대규모 serving에서 예측 가능

---

## 16. 운영용 산정 절차

```
Step 1. 모델이 GPU에 올라가는지 확인
  VRAM_required = resident_weight
                + max_concurrent_sequences × context_len × KV_per_token
                + runtime_overhead + workspace + fragmentation_margin

Step 2. Decode roofline 상한 계산
  TPS_roof = N × aggregate_HBM_BW / (W + N × S × K)
  (DP: GPU별 합산 / TP: aggregate HBM 사용 + communication overhead 추가)

Step 3. Effective utilization factor 적용
  TPS_cap = TPS_roof × η

Step 4. Prefill budget 별도 계산
  Prefill load = requests/sec × avg_input_tokens

Step 5. CPU/RAM/PCIe 병목 점검
Step 6. Load test로 보정
```

**η 권장 초깃값:**

| 상황 | η |
|------|---|
| 검증 전 보수 산정 | 0.50~0.65 |
| 최적화된 vLLM/TensorRT-LLM, 단순 workload | 0.65~0.80 |
| 벤치마크 친화 workload | 0.80~0.90 |
| p95 SLA 운영 산정 | 보통 0.65 이하에서 시작 |

**Step 6 최종 기준** (평균 TPS가 아님): p50/p95/p99 TTFT, p50/p95/p99 ITL, sustained output TPS, request timeout rate, KV cache eviction rate, GPU memory headroom, CPU scheduler saturation.

---

## 17. 실전 의사결정 요약

- **70B INT4를 H100 80GB에?** 가능. weight ~35GB, GQA FP16 KV context 2048 기준 사용자당 ~0.67GB. 50명대 동시 decode부터 overhead/SLA 실측 필수.
- **70B FP16을 H100 80GB 단일장에?** 불가능. weight만 ~140GB. H200 141GB도 runtime+KV 고려 시 단일장 실서비스 부족.
- **RAM이 크면 VRAM 부족을 대체?** 불가. offload는 PCIe-bound 또는 CPU/RAM-bound가 되어 HBM-resident serving보다 크게 느림.
- **H100 2장으로 100명×50 TPS 보장?** 단순 DP/FP16 KV로는 어려움. 후보는 2× H100 TP=2 + NVLink + FP8 KV. prefill 부하와 FP8 KV regression test 필요.
- **DP vs TP 우선순위?** 모델이 단일 GPU에 들어가면 DP 먼저(운영 단순, 장애 격리, autoscaling). DP로 부족하거나 모델이 안 들어가면 TP — 단 NVLink/NVSwitch 전제.
- **H200 vs B200?** H200은 VRAM/bandwidth가 커 long-context·batch serving 유리. B200은 양쪽 모두 크게 개선되어 단일 GPU 70B serving headroom이 훨씬 좋음. 단 sparse/dense·FP4/FP8/FP16 지표 구분 필수.

---

## 18. 최종 명제 검증

| 명제 | 판정 | 정확한 해석 |
|------|------|------------|
| TPS는 FLOPS가 아니라 대역폭이다 | **조건부 참** | dense low/medium-batch decode에서 HBM BW가 1차 병목 |
| Prefill도 대역폭 문제다 | **대체로 거짓** | prefill은 큰 GEMM/attention으로 compute/attention-bound 가능 |
| 배칭하면 TPS가 N배 된다 | **초기 구간 참** | KV cache와 scheduler/latency가 곧 병목 |
| KV cache는 모델마다 비슷하다 | **거짓** | MHA/GQA/MQA에 따라 크게 달라짐 |
| Llama 3 70B KV는 2.62MB/token | **거짓** | GQA 기준 FP16 약 0.328MB/token |
| 70B FP16은 H100 80GB 단일장 가능 | **거짓** | weight만 ~140GB |
| H200 단일장이면 70B FP16 실서비스 가능 | **대체로 거짓** | 141GB는 weight만 간신히 |
| RAM offload로 VRAM 부족 해결 | **성능상 위험** | PCIe 또는 CPU/RAM roofline이 병목 |
| H100 1,979 TFLOPS를 dense에 그대로 | **거짓** | sparsity 수치, dense는 ~989.5 TFLOPS |
| TP N개면 TPS N배 | **조건부 참** | NVLink/NVSwitch와 통신 overhead 관리 필요 |
| FP8 KV는 무조건 안전하다 | **거짓** | calibration과 regression test 필요 |
| MoE는 dense와 같은 공식 | **거짓** | VRAM은 total params, decode bandwidth는 active params |

---

## 19. 결론

> **Decode serving은 "HBM에 resident한 effective model bytes와 KV cache bytes를 얼마나 빨리 읽느냐"의 문제이고, 실서비스는 그 위에 CPU scheduler, PCIe, System RAM, interconnect, prefill interference, tail latency를 얹은 문제다.**

좋은 capacity planning 문서는 반드시 네 가지를 분리한다:

1. 모델이 VRAM에 올라가는가?
2. decode roofline 상한은 얼마인가?
3. prefill load가 TTFT/ITL을 얼마나 잠식하는가?
4. p95 SLA 기준으로 이론 상한의 몇 %를 capacity로 인정할 것인가?

**70B급 운영 원칙:**
- H100 80GB 단일장: 70B INT4 + GQA + FP16/FP8 KV 실측 검증
- 2× H100: TP=2를 쓰려면 NVLink 사실상 전제
- 5,000 TPS급 SLA: 2× H100 TP=2 + NVLink + FP8 KV가 최소 후보
- DP-only 선호: 4× H100 DP + FP8 KV 이상
- FP16 KV 고수: GPU 수 급증
- Long-context/고동시성: H200/B200 가치 상승
- Offload: 기능적 fallback이지 고성능 serving 전략 아님

**핵심 공식 3개:**

```
KV_bytes/token
  = 2 × layers × n_kv_heads × head_dim × bytes

Total_TPS_roof
  ≈ N × aggregate_HBM_BW / (W + N × context_len × KV_bytes/token)

TPS_capacity
  ≈ TPS_roof × effective_utilization_factor
```

이 세 공식에 **VRAM 적재성, prefill budget, PCIe/NVLink 통신, CPU/RAM 병목**을 함께 넣으면 LLM 서버 산정 오류의 대부분을 피할 수 있다.

---

## 참고문헌

[1] S. Williams et al., "Roofline: an insightful visual performance model," *CACM*, 2009. doi:10.1145/1498765.1498785

[2] R. Pope et al., "Efficiently Scaling Transformer Inference," *MLSys 2023* (Outstanding Paper). arXiv:2211.05102

[3] T. Dao, "FlashAttention-2: Faster Attention with Better Parallelism," *ICLR 2024*. arXiv:2307.08691

[4] G.-I. Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models," *OSDI 2022*.

[5] W. Kwon et al., "Efficient Memory Management for LLM Serving with PagedAttention," *SOSP 2023*. arXiv:2309.06180

[6] J. Ainslie et al., "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints," *EMNLP 2023*. arXiv:2305.13245

[7] Meta AI, "Llama 3 Model Card," 2024 — Llama 3 70B GQA(n_kv_heads=8) 구조.

[8] Mistral AI, "Mixtral of Experts," 2024 — 총 46.7B / token당 12.9B active params.

[9] NVIDIA, "H100 / H200 / B200 Tensor Core GPU Architecture Whitepapers" (Dense vs Sparse 스펙).

[10] vLLM Documentation, "FP8 KV Cache" — calibration 기반 scale 산정 권장.

---

## 관련 노트
- [[LLM-GPU-Complete-Analysis]] — **v2**: GPU 중심 분석 (이 문서의 전신)
- [[LLM-TPS-CUDA-Analysis-Report]] — 학술 논문 인용 중심 보고서
- [[LLM-Inference-Bandwidth]] — TPS 공식 원본 분석
- [[CUDA-Role-in-LLM]] — CUDA 최적화 기법 상세
- [[Inference-Optimization]] — vLLM, FlashAttention, 양자화 구현
- [[Transformer-Architecture]] — GQA, MoE 아키텍처
- [[../hardware/gpu-workstation/DGX-Spark]] — H100/B200 하드웨어 스펙
- [[../hardware/gpu-workstation/Apple-M5-Max]] — 통합 메모리 아키텍처
