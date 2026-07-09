---
tags:
  - ai
  - vllm
  - inference
  - serving
  - llmops
  - production
created: 2026-06-17
---

# vLLM — 프로덕션 LLM 추론 서버

> [!summary] 한 줄 요약
> vLLM은 **PagedAttention**(KV cache 페이징)과 **Continuous Batching**(iteration 단위 스케줄링)으로 GPU를 최대한 채우는 오픈소스 추론 엔진이다. OpenAI 호환 API를 제공해 [[GPU-Inference-Decision-Guide|의사결정 가이드]]에서 고른 GPU의 이론 처리량을 실제로 끌어내는 표준 서빙 스택이다. [Kwon et al., SOSP 2023]

---

## 1. 왜 vLLM인가 — naive 서빙과의 차이

```
naive PyTorch / HF generate:
  Static batching → 배치 내 한 요청 끝날 때까지 GPU 유휴
  KV cache 연속 할당 → 60~80% 단편화
  → 이론 대역폭의 20~30%만 사용

vLLM:
  Continuous Batching → 매 step 완료 슬롯 즉시 교체, GPU 가동률 ↑
  PagedAttention → KV cache 블록 단위 관리, 단편화 ~4%
  → 이론 대역폭의 70~90% 도달, 처리량 2~4× (vs FasterTransformer)
```

vLLM이 하는 일은 [[CUDA-Role-in-LLM]]의 "이론 천장에 실제로 도달"과 [[LLM-System-Performance-Analysis|v3]]의 "η(실효 이용률)를 0.7~0.8로 올리기"에 정확히 대응한다.

---

## 2. 설치 & 기본 실행

```bash
# 설치 (CUDA 환경)
pip install vllm

# OpenAI 호환 서버 기동 (가장 흔한 사용)
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --quantization awq \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90

# Python 오프라인 배치 추론
python -c "
from vllm import LLM, SamplingParams
llm = LLM(model='Qwen/Qwen2.5-32B-Instruct-AWQ', quantization='awq')
out = llm.generate(['안녕하세요'], SamplingParams(temperature=0.7, max_tokens=256))
print(out[0].outputs[0].text)
"
```

---

## 3. OpenAI 호환 API

vLLM 서버는 OpenAI SDK·기존 클라이언트와 그대로 호환된다 → [[REST-API]]·[[Spring-Cloud-Gateway]] 뒤에 바로 붙일 수 있다.

```bash
# 채팅 (스트리밍)
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.3-70B-Instruct",
    "messages": [{"role": "user", "content": "RAG란?"}],
    "stream": true,
    "max_tokens": 512
  }'
```

```java
// Spring AI / OpenAI 클라이언트에서 base-url만 vLLM으로 교체
// application.yml
spring:
  ai:
    openai:
      base-url: http://vllm-service:8000
      api-key: dummy            # vLLM은 --api-key 미설정 시 검증 안 함
      chat:
        options:
          model: meta-llama/Llama-3.3-70B-Instruct
```

---

## 4. 핵심 실행 옵션 (capacity 직결)

| 옵션 | 역할 | v3 연결 |
|------|------|---------|
| `--tensor-parallel-size N` | TP로 N장에 모델 분할 | 한 장에 안 들어갈 때. NVLink 권장 (v3 13절) |
| `--pipeline-parallel-size N` | 레이어를 N단계로 분할 (멀티노드) | 초대형 모델 |
| `--quantization awq\|gptq\|fp8` | 가중치 양자화 | 모델크기↓ → TPS↑ + 적재 (게이트 통과) |
| `--kv-cache-dtype fp8` | KV cache FP8 압축 | 동시 사용자 2× (v3 7절) |
| `--gpu-memory-utilization 0.90` | VRAM 사용 상한 | 0.80~0.90 권장 (overhead 여유) |
| `--max-model-len 8192` | 최대 context 길이 | KV cache 상한 결정 |
| `--max-num-seqs 256` | 최대 동시 배치 크기 N | 처리량 vs latency 트레이드오프 |
| `--enable-chunked-prefill` | prefill을 chunk로 분할 | TTFT/ITL 균형 (v3 15절) |
| `--enable-prefix-caching` | 공통 prefix KV 재사용 | system prompt·few-shot 반복 시 |

```bash
# 다수 사용자 70B 서빙 예시 (v3 14절 옵션 D 구성)
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --quantization awq \              # 70B → ~35GB
  --kv-cache-dtype fp8 \            # 동시 사용자 2×
  --tensor-parallel-size 2 \       # 2× H100 + NVLink
  --max-model-len 8192 \
  --max-num-seqs 256 \
  --enable-chunked-prefill \
  --gpu-memory-utilization 0.90
```

---

## 5. PagedAttention & Continuous Batching

```
[PagedAttention]
KV cache를 고정 크기 block으로 분할, OS 가상메모리 페이징처럼 관리
→ 비연속 물리 메모리 허용 → 단편화 60~80% → ~4%
→ prefix sharing (동일 system prompt 블록 공유)

[Continuous Batching]
매 decode iteration마다 완료된 시퀀스 슬롯에 대기 요청 즉시 삽입
→ GPU 유휴 최소화
→ FasterTransformer 대비 최대 36.9× 처리량 [Yu et al., OSDI 2022]
```

이 둘이 vLLM의 핵심이며 [[Inference-Optimization]]·[[LLM-System-Performance-Analysis|v3]] 10절에서 상술한 메커니즘이다.

---

## 6. 양자화 지원

```
가중치 양자화:
  AWQ   → --quantization awq    (품질 우수, 추론 빠름, 가장 권장)
  GPTQ  → --quantization gptq
  FP8   → --quantization fp8    (H100+ 네이티브 FP8)
  INT8  → W8A8 (compressed-tensors)

KV cache 양자화:
  --kv-cache-dtype fp8          # 동시 사용자 2×, 정확도 <0.3%
  --kv-cache-dtype fp8_e5m2 / fp8_e4m3

주의 (v3 7절):
  FP8 KV는 무조건 안전하지 않음 → calibration·regression test 필요
```

> Apple/CPU에서는 vLLM 대신 [[Inference-Optimization|llama.cpp/MLX]]를 쓴다. vLLM은 CUDA(+ROCm 일부) 중심이다.

---

## 7. 분산 배포 — TP / PP / DP

```
[단일 노드 TP] 한 장에 안 들어가는 모델
  --tensor-parallel-size 4      # 4 GPU, NVLink/NVSwitch 권장

[멀티 노드 PP+TP] 초대형 모델 (405B 등)
  --tensor-parallel-size 8 --pipeline-parallel-size 2   # 16 GPU
  + Ray 클러스터

[데이터 병렬 DP] 한 장에 들어가는 모델 (70B INT4, ≤32B)
  → vLLM 인스턴스를 카드마다 독립 기동 + 앞단 로드밸런서
  → K8s Deployment replicas + Service
  → NVLink 불필요, 처리량 ×N (v3 12절)
```

> 의사결정: 모델이 한 장에 들어가면 **DP(인스턴스 복제)** 우선, 안 들어가면 **TP(+NVLink)**. → [[GPU-Inference-Decision-Guide]]

---

## 8. 성능 튜닝 레버

```
Chunked Prefill (--enable-chunked-prefill)
  긴 prompt를 chunk로 쪼개 decode와 interleave → TTFT/ITL 균형

Prefix Caching (--enable-prefix-caching)
  공통 system prompt·few-shot의 KV를 재사용 → 반복 요청 prefill 절감

Speculative Decoding (--speculative-model)
  draft 모델로 복수 토큰 추측 → 2~3× decode [Leviathan et al., ICML 2023]
  vllm serve <target> --speculative-model <draft> --num-speculative-tokens 5

CUDA Graph (기본 활성)
  decode 반복 그래프 캡처 → CPU launch overhead 감소
  --enforce-eager 로 비활성 (디버깅 시)
```

---

## 9. 프로덕션 운영

### 9.1 Prometheus 메트릭

```
vLLM은 /metrics 엔드포인트로 Prometheus 메트릭 노출:
  vllm:num_requests_running        # 현재 처리 중 요청
  vllm:num_requests_waiting        # 대기 큐 (backpressure 신호)
  vllm:gpu_cache_usage_perc        # KV cache 사용률 (포화 임박 감지)
  vllm:time_to_first_token_seconds # TTFT (p95 SLA)
  vllm:time_per_output_token_seconds # ITL
  vllm:e2e_request_latency_seconds
```

→ [[AOP-Logging]]·[[Resilience4j]]의 Grafana 대시보드 패턴과 동일하게 연동. `gpu_cache_usage_perc`가 지속 90%+면 동시 사용자 한계 도달 신호.

### 9.2 Kubernetes 배포 (DP replica)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama70b
spec:
  replicas: 4                      # DP: 카드 4장 = replica 4 (v3 12절)
  template:
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
          - "--model=meta-llama/Llama-3.3-70B-Instruct"
          - "--quantization=awq"
          - "--kv-cache-dtype=fp8"
          - "--max-model-len=8192"
          - "--gpu-memory-utilization=0.90"
        resources:
          limits:
            nvidia.com/gpu: 1       # replica당 1 GPU (RTX PRO 6000 96GB)
        ports:
          - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-svc
spec:
  selector: { app: vllm-llama70b }
  ports: [{ port: 8000, targetPort: 8000 }]
```

> TP 구성이면 replicas=1 + `nvidia.com/gpu: N` + `--tensor-parallel-size N`, 그리고 NVLink 노드에 스케줄. → [[Kubernetes]]·[[Helm]]

---

## 10. 트러블슈팅

| 증상 | 원인 | 대응 |
|------|------|------|
| OOM 기동 실패 | weight+KV가 VRAM 초과 | `--gpu-memory-utilization↓`, 양자화, `--max-model-len↓` |
| TTFT 폭증 | 긴 prompt가 decode 방해 | `--enable-chunked-prefill` |
| 처리량 낮음 | 배치 작음 | `--max-num-seqs↑`, KV FP8로 여유 확보 |
| `num_requests_waiting` 누적 | capacity 초과 | replica 추가(DP) 또는 GPU 증설 |
| TP에서 느림 | NVLink 없음 (PCIe 병목) | NVLink 노드 배치, 또는 한 장 적재 후 DP |

---

## 11. 핵심 정리

```
1. vLLM = PagedAttention + Continuous Batching으로 η를 0.7~0.8로 끌어올리는 엔진
2. OpenAI 호환 API → Spring/기존 클라이언트에 base-url만 교체해 연동
3. 핵심 옵션: --quantization, --kv-cache-dtype, --tensor-parallel-size, --max-num-seqs
4. 한 장 적재 → DP replica / 안 들어감 → TP(+NVLink)
5. /metrics의 gpu_cache_usage_perc·waiting로 capacity 한계 모니터링
```

---

## 참고문헌
[1] W. Kwon et al., "Efficient Memory Management for LLM Serving with PagedAttention," *SOSP 2023*. arXiv:2309.06180
[2] G.-I. Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models," *OSDI 2022*.
[3] Y. Leviathan et al., "Fast Inference from Transformers via Speculative Decoding," *ICML 2023*. arXiv:2211.17192
[4] vLLM Documentation — https://docs.vllm.ai

---

## 관련 노트
- [[Inference-Optimization]] — KV Cache·양자화·PagedAttention 이론
- [[Prompt-Caching]] — vLLM prefix caching = 입력 KV 재사용(TTFT↓), 그 원리·운영
- [[GPU-Inference-Decision-Guide]] — GPU 선택 (vLLM 배포 대상)
- [[LLM-System-Performance-Analysis]] — v3: capacity 산정, η, DP/TP
- [[LLMOps]] — 모델 게이트웨이·비용·모니터링
- [[RAG-Chatbot-E2E]] — vLLM 백엔드 + Spring AI RAG
- [[Kubernetes]] · [[Helm]] — 서빙 배포 인프라
