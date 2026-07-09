---
tags:
  - ai
  - llm
  - moc
  - index
created: 2026-06-15
updated: 2026-06-16
---

# AI / LLM Knowledge Base — MOC

> LLM 애플리케이션 개발 지식 정리. (백엔드는 [[../programming/spring/_index|Spring KB]] 참고)

## 기초 수학 / ML 이론
- [[Math-Foundations]] — 행렬 곱연산, 부동소수점(FP32/BF16/FP8), 선형대수, 경사 하강법
- [[ML-Fundamentals]] — Train/Validation/Test 분할, 과적합, Regression, Classification, Bayesian

## 아키텍처 & 원리
- [[Transformer-Architecture]] — Self-Attention, MHA, GQA, Positional Encoding (RoPE), Causal Masking, MoE, Scaling Laws
- [[LLM]] — 대규모 언어 모델 기초: 토큰, 컨텍스트, 모델 선택, 호출 패턴
- [[Open-Source-Models]] — Llama 3 / Mistral / **Hermes** / Qwen / DeepSeek — 모델 특성 및 선택 가이드
- [[VLM]] — Vision Language Model: ViT 인코더, 이미지 토큰, InternVL2 / Llama 3.2 Vision / Qwen2.5-VL
- [[SLLM]] — Small LLM: Phi-4 / Gemma 2 / 온디바이스 배포, Knowledge Distillation, 용도별 선택 가이드

## 학습 기법
- [[Fine-Tuning]] — SFT, LoRA/QLoRA, RLHF, DPO, SimPO, ORPO (LLM)
- [[Parametric-Knowledge-Editing]] — 파라메트릭 지식 편집·핫스왑(DMoE fact hot-swap): FFN=지식저장소·ROME/MEMIT·라우터/지역화 난제, RAG와 상보 ⭐
- [[YOLO-Finetuning]] — 커스텀 객체 탐지 파인튜닝: 데이터 분포 진단, 증강, 작은 객체·스케일 편향 대응(SAHI/P2/imgsz), 현장 케이스
- [[YOLO-Detection-Pipeline]] — 동작 원리(그리드·objectness·멀티스케일·label assignment·NMS·DFL) + cascade 파이프라인(사람→머리crop→안전모 PPE)
- [[Object-Detection-Metrics]] — 평가 지표: IoU·Precision/Recall·AP·mAP, mAP@50 vs @50-95, 크기별 진단

## 추론 최적화
- [[Inference-Optimization]] — KV Cache, 양자화(GPTQ/AWQ/GGUF), Speculative Decoding, Continuous Batching, Flash Attention, vLLM
- [[vLLM]] — **프로덕션 추론 서버**: PagedAttention·Continuous Batching, OpenAI 호환 API, 양자화·TP/DP 옵션, K8s 배포, 메트릭 운영
- [[LLM-Inference-Bandwidth]] — TPS의 물리학: 대역폭/모델크기, Prefill vs Decode 병목 차이, Roofline Model, 하드웨어별 실측
- [[CUDA-Role-in-LLM]] — CUDA의 역할: 이론 천장(대역폭) vs 실제 도달률, Flash Attention·Fused Kernel·양자화 커널·CUDA Graph

### 🖥️ GPU 성능 분석 & 하드웨어 선택 (시리즈)
> 추천 진입점: 실무 선택은 **[[GPU-Inference-Decision-Guide|의사결정 가이드]]**, 이론 깊이는 **[[LLM-System-Performance-Analysis|v3 시스템 보고서]]**
- [[LLM-TPS-CUDA-Analysis-Report]] — **학술 보고서**: TPS=대역폭/모델크기 명제 검증, 논문 10편 인용
- [[LLM-GPU-Complete-Analysis]] — **v2**: 추론속도·다중사용자 처리량·학습 하드웨어, GQA KV Cache, Dense/Sparse TFLOPS
- [[LLM-System-Performance-Analysis]] — **v3 시스템 보고서**: GPU+CPU+RAM+PCIe 통합, Capacity planning, DP vs TP, NVLink, SLA 산정
- [[GPU-Inference-Decision-Guide]] — **실전 의사결정 가이드**: 단일/다수 사용자 × 모델크기 트리, 카드 카탈로그, 함정, 치트시트
- 하드웨어: [[../hardware/gpu-workstation/_index|GPU Workstation]] · [[../hardware/gpu-workstation/Apple-Silicon-Inference|Apple Silicon 추론(코어/NPU/대역폭)]] · [[../hardware/edge-ai/Jetson|Jetson 엣지 AI]] 절차

## 임베딩 & 검색
- [[Embedding]] — 임베딩 모델 선택, 코사인 유사도, HNSW, pgvector, 하이브리드 검색
- [[RAG]] — 검색 증강 생성: 청킹, 벡터 DB, 리랭킹, Spring AI 구현
- [[RAG-Ingestion-Pipeline]] — 인입 파이프라인 운영(상류 천장): 파싱 품질·증분 동기화·삭제 전파(tombstone)·메타데이터·실시간 vs 배치 ⭐
- [[Chunking-Strategies]] — 청킹 전략 심화: fixed/recursive/semantic·Contextual Retrieval(Anthropic)·late chunking·parent-document ⭐
- [[Reranking-Retrieval]] — 리랭킹·2단계 검색: bi vs cross-encoder·top-N→top-k·bge/Cohere reranker·RRF 하이브리드 융합 ⭐
- [[Query-Transformation]] — 쿼리 변환(검색 전): HyDE·multi-query·step-back·decomposition·라우팅 ⭐
- [[RAG-Security]] — RAG 보안: 간접 프롬프트 인젝션·코퍼스 포이즈닝·데이터 유출·검색콘텐츠 비신뢰 방어 ⭐
- [[RAG-Chatbot-E2E]] — E2E 예시: 사내 문서 챗봇 구축 (Spring AI + pgvector + LLMOps 모니터링)
- [[RAG-vs-Knowledge-Systems]] — "LLM Wiki도 RAG인가?" — 파라메트릭 vs 비파라메트릭, CAG, GraphRAG, Agentic RAG 전체 지도
- [[CAG-Cache-Augmented-Generation]] — 검색 없는 지식 주입: 전체 컨텍스트+KV 캐시·성립 조건·이 볼트 840K 실측·무효화 함정 ⭐
- [[Multimodal-RAG]] — 시각 문서 검색(ColPali/ColQwen): 파싱→텍스트 vs OCR-free 페이지 임베딩·late-interaction ⭐
- [[Agentic-RAG]] — 반복·교정 검색(Self-RAG reflection 토큰·CRAG grader→웹 폴백·multi-hop·에이전트 결정·루프 폭주 한도) ⭐
- [[Citation-Attribution]] — 인용·출처 귀속: 인라인 인용 생성·스팬 귀속·인용 환각·사후 검증(NLI/judge)·트레이드오프 ⭐
- [[RAG-Failure-Debugging]] — 실패 디버깅 플레이북: 4단계 책임 귀속(검색 누락/순위/컨텍스트/생성)·gold chunk 강제 주입으로 검색↔생성 이분·context_recall vs faithfulness ⭐
- [[Structured-Data-RAG]] — 구조화 데이터 RAG(text-to-SQL): 집계·필터 질의는 벡터 검색이 구조적 실패·라우팅·스키마 링킹·SQL 검증 가드·환각 SQL 함정·문서+SQL 하이브리드 ⭐
- [[Conversational-RAG]] — 대화형·멀티턴 RAG: 후속 질문을 이력으로 standalone 재작성(contextualization)·재검색 vs 재사용 판단·이력 관리(요약/윈도우)·화제 전환 오염 함정 ⭐
- [[Self-Query-Retrieval]] — Self-Query 검색: NL 질의를 {메타 필터}+{의미 질의}로 자동 분해(스키마 인지)·pre vs post filter·over-filtering→0건 함정·ACL 필터와 분리 ⭐
- [[Code-RAG]] — 코드베이스 RAG: AST 기반 청킹(함수/클래스 경계)·심볼 인덱스/콜그래프(정의↔사용)·코드 임베딩·의존성 확장·동명 심볼/테스트 노이즈 함정 ⭐
- [[Context-Assembly-Compression]] — 리랭킹↔생성 사이 레버: 배치 순서(lost-in-middle: head·tail)·토큰 예산 배분·프롬프트 압축(LLMLingua)·중복 병합 ⭐
- [[GraphRAG]] — 그래프 기반 RAG: 엔티티/관계·Leiden 커뮤니티·global vs local 검색·인덱싱 비용 트레이드오프
- [[LLM-Wiki]] — 검색 레이어 없는 KB 구축 가이드: 구조=검색엔진(인덱스·설명·링크)·progressive disclosure·Wiki 우선 하이브리드 ⭐
- [[RAG-Latency-Reality]] — "RAG는 느리다"의 실체: 병목은 LLM 생성(98%), 구조화의 가치는 속도가 아닌 정확도
- [[LLM-Auto-Tagging-Pipeline]] — 색인 시점 의미 강화: 인입 시 LLM이 태그·별칭·요약·Doc2Query 질문 추출→통제 어휘(taxonomy)+사람 승인으로 태그 폭발 방지, 임베딩 없이 키워드 검색으로 의미 매칭 ⭐
- [[KB-Database-Schema]] — 사내 KB DB 스키마: 진실원천 정규화(태그/taxonomy/권한/버전 FK)+검색 비정규화(tsv materialize)+jsonb, 그룹기반 ACL·RLS, 과정규화 경계 ⭐
- [[KB-Access-Control]] — KB 권한 설계(ACL): 주체/자원/권한, 그룹기반 RBAC, 검색 pre-filter 철칙, PG진실원천+ES 미러링, RLS 방어심층, 권한 drift·캐시 누출 함정 ⭐

## 에이전트
> 🐍 **Hermes 에이전트 프레임워크 전반은 별도 폴더로 분리** → [[ai/hermes/_index|Hermes 에이전트 MOC]] (개요·만드는 법·스킬·플러그인·메모리·오케스트레이션·배포·설정·실사용 예시)
> 🦞 **OpenClaw 자율 런타임도 별도 폴더로 분리** → [[ai/openclaw/_index|OpenClaw MOC]] (개요·아키텍처·백엔드 모델·하이브리드 라우팅·코딩 팀 응용)

- [[Harness-Engineering]] — **하네스 엔지니어링**: Agent=Model+Harness, "모델보다 하네스"(같은 모델 46%↔80%), 에이전트 문제는 시스템 설계 문제 ⭐
- [[Agent-Harness]] — 하네스 구현: 에이전트 루프, 툴 호출, 컨텍스트 관리, 가드레일
- [[MCP-Skill]] — Model Context Protocol, Skill 시스템, 커스텀 MCP 서버
- [[Autonomous-Agent-Runtimes]] — 자율 에이전트 런타임 비교: OpenClaw vs Hermes Agent, 모델 vs 런타임 계층 (→ [[ai/openclaw/_index|OpenClaw]] · [[ai/hermes/_index|Hermes]])
- [[Agent-Security]] — 에이전트 보안·샌드박싱: 최소권한·격리·승인 게이트·인젝션/도구남용 방어·감사·자가개선 통제
- [[Claude-Code-Headless]] — `claude -p` 헤드리스 모드 레퍼런스: 옵션(output-format·permission-mode·resume·mcp-config)·JSON 결과·구독 재사용(무과금)·한도/타임아웃 함정
- [[Claude-Tag]] — **Claude Tag**(2026-06, Anthropic): Slack 채널 팀원 AI, 멀티플레이어 공유 컨텍스트 — Hermes 게이트웨이의 관리형 대안 분석
- [[Tool-Search]] — 툴 과부하 대응 점진적 공개: 3 브리지 툴(search→describe→call)·auto 임계·코어 제외·브리지 언랩
- → 세부: OpenClaw [[ai/openclaw/_index|OpenClaw MOC]] · Hermes [[ai/hermes/_index|Hermes MOC]] 참조

## 프롬프트 & 평가
- [[Prompt-Engineering]] — 프롬프트 설계 (Few-Shot·CoT·구조화·캐싱 요약)
- [[Prompt-Caching]] — 프롬프트 캐싱 전용: 입력 KV 재사용, TTFT↓·비용 90%↓, vLLM/API, ≠응답캐싱, CAG
- [[Evaluation]] — Evals, RAGAS (Faithfulness/Recall/Precision/Relevancy), LLM-as-Judge, CI/CD 통합

## 프로덕션 운영
- [[LLMOps]] — 모델 게이트웨이, 비용 최적화, 환각 탐지, A/B 테스트, 프롬프트 인젝션 방어
- [[Local-LLM-API-Deployment]] — Mac 로컬 LLM(Hermes 등)을 OpenAI 호환 API로 배포, 개발↔프로덕션 이전 경로

## Java AI 추론
- [[DJL]] — Deep Java Library: YOLO 객체 감지, 버전별(v8/v9/v11) + n/s/m/l/x 크기 비교, ONNX 로드, CCTV 연동

## 워크플로우 자동화
- [[N8N-Automation]] — n8n(비즈니스 자동화), LangChain(LLM 체인·RAG), LangGraph(상태 기반 멀티 에이전트·HITL)

## 하드웨어
- [[../hardware/gpu-workstation/DGX-Spark|DGX-Spark]] — NVIDIA GB10 Grace Blackwell, 1 PetaFLOP, CUDA 생태계
- [[../hardware/gpu-workstation/Apple-M5-Max|Apple-M5-Max]] — M5 Max 40코어 GPU, ~600 GB/s, mlx-lm 완전 가이드
- [[../hardware/gpu-workstation/GPU-Specs|GPU-Specs]] — GeForce 4080~5090, A100/H100/H200/B200, L40S, RTX PRO 종합 스펙
- [[../hardware/edge-ai/Jetson|Jetson]] — NVIDIA Jetson 엣지 AI (Orin Nano/AGX Orin/AGX Thor), 로보틱스·CCTV·온디바이스 LLM

## 관점 · 인사이트
- [[AI-Engineering-Future]] — AI 시대 소프트웨어 엔지니어링: 코딩은 상향평준화, 차별화는 도메인 지식·설계·기획으로 수렴

---

## 🧭 한눈에 — LLM 앱의 3가지 티어
```
1. 단일 호출   : 분류/요약/추출/QA          → LLM API 1회
2. 워크플로우  : 코드로 제어하는 다단계 + 툴  → LLM + tool use (내가 루프 제어)
3. 에이전트    : 모델이 스스로 경로 결정       → agentic loop ([[Agent-Harness]])
```
> 원칙: **가장 단순한 티어부터.** 대부분은 1~2티어로 충분하다. 3티어는 열린 탐색이 꼭 필요할 때만.

## 🔑 기본값 (이 KB 기준, 2026-06)
- 모델: **Claude Opus 4.8** (`claude-opus-4-8`), 1M 컨텍스트
- 오픈소스 로컬: **Hermes-3 70B** (에이전트), **Llama 3.3 70B** (일반), **QwQ-32B** (추론)
- 추론 서버: **[[vLLM]]** (프로덕션), **mlx_lm.server** (M5 Max 로컬), **Ollama** (개발)
- 임베딩: **bge-m3** (한국어 오픈소스), **voyage-3** (Claude 연동)

## 관련 (Spring 연계)
- [[REST-API]] · [[Spring-Cloud-Gateway]] · [[Redis]](임베딩/응답 캐시) · [[Kafka]](비동기 추론)
