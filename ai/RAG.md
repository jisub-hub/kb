---
tags:
  - ai
  - rag
  - llm
  - embedding
  - vector-db
created: 2026-06-15
---

# RAG (Retrieval-Augmented Generation)

> [!summary] 한 줄 요약
> LLM이 모르거나 최신이 아닌 정보를, **외부 지식베이스에서 관련 문서를 검색(retrieve)** 해 프롬프트에 넣어준 뒤 답변을 **생성(generate)** 하게 하는 패턴. 환각을 줄이고, 모델 재학습 없이 도메인/사내 지식을 활용한다.

---

## 1. 왜 RAG인가
- LLM은 **학습 컷오프 이후·비공개 데이터**를 모른다.
- 모든 걸 컨텍스트에 넣는 것은 비용·한계상 비현실적.
- 파인튜닝은 비싸고 갱신이 느림.
→ **필요한 조각만 검색해 그때그때 주입.** 출처를 함께 주면 **인용·검증** 가능.

RAG는 사전학습된 파라메트릭 메모리와 비파라메트릭 검색 메모리를 결합해 지식 집약적 NLP 태스크에서 순수 생성 모델 대비 더 사실적이고 다양한 응답을 생성한다 [Lewis et al., 2020].

```
[질문] ─► 임베딩 ─► 벡터 검색 ─► 관련 청크 top-k ─┐
                                                  ├─► 프롬프트(질문+근거) ─► LLM ─► 답변(+인용)
[지식베이스 문서] ─► 청킹 ─► 임베딩 ─► 벡터 DB ────┘   (사전 인덱싱, offline)
```

---

## 2. 파이프라인 2단계

### 2.1. 인덱싱 (offline, 사전 작업)
1. **로드**: 문서 수집(PDF/HTML/DB/노션 등)
2. **청킹(chunking)**: 문서를 적절한 크기로 분할 (핵심 품질 요인)
3. **임베딩**: 각 청크를 벡터로 변환
4. **저장**: 벡터 DB에 (벡터 + 원문 + 메타데이터) 적재

### 2.2. 질의 (online, 요청 시)
1. 질문을 **임베딩**
2. 벡터 DB에서 **유사도 검색**(top-k)
3. (선택) **리랭킹**으로 정밀도↑
4. 검색 결과를 **프롬프트에 주입** + 질문
5. LLM이 **근거 기반 답변** 생성 (출처 인용)

---

## 3. 핵심 구성요소

### 청킹(Chunking) — 품질의 절반
- 너무 크면: 노이즈↑·비용↑·검색 정밀도↓. 너무 작으면: 문맥 손실.
- 전략: 고정 크기 + **오버랩**(경계 문맥 보존), 문단/문장/마크다운 헤더 기반, 시맨틱 청킹.
- 경험치: 500~1000 토큰 + 10~20% 오버랩에서 시작 → 평가로 튜닝.
- **메타데이터**(출처/섹션/날짜) 함께 저장 → 필터링·인용에 필수.

### 임베딩(Embedding)
- 텍스트 → 의미를 담은 고차원 벡터. 의미가 가까우면 벡터도 가깝다(코사인 유사도).
- 질의·문서에 **동일 임베딩 모델** 사용. 모델 교체 시 **전체 재색인** 필요.

### 벡터 DB
| 옵션 | 특징 |
|------|------|
| **pgvector** (PostgreSQL 확장) | 기존 [[RDS-vs-Self-Hosted|RDS/PG]]에 통합. 운영 단순, 중소 규모 ⭐ → [[pgvector|pgvector 설정 가이드]] |
| **Qdrant / Weaviate / Milvus** | 전용 벡터 DB. 대규모·고성능·필터링 풍부 |
| **Elasticsearch / OpenSearch** | 키워드+벡터 하이브리드 |
| **Redis (Vector)** | [[Redis]] 모듈로 저지연 검색 |
- 인덱스: **HNSW**(근사 최근접, ANN)가 사실상 표준 — 정확도/속도 균형.

---

## 4. 검색 품질 향상 기법

### 기본 기법
- **하이브리드 검색**: 벡터(의미) + BM25(키워드) 결합 → 고유명사·코드에 강함 [Robertson & Zaragoza, 2009].
- **리랭킹(reranking)**: top-50을 가져와 cross-encoder로 재정렬 후 top-5만 사용.
- **메타데이터 필터링**: 기간·권한·문서종류로 사전 좁히기.
- **인용/근거 표시**: 답변에 출처 청크를 함께 → 검증 가능 + 환각 억제.

### 고급 기법

#### HyDE (Hypothetical Document Embeddings)
질문을 직접 임베딩하지 않고, **LLM이 가상의 답변을 생성 → 그 답변을 임베딩** [Gao et al., 2022].
```
[일반 검색]  질문 임베딩 → 문서 임베딩 공간에서 검색
              질문과 답변의 표현 방식이 달라 미스매치 발생

[HyDE]       질문 → LLM → "가상 답변" → 가상 답변 임베딩 → 검색
              가상 답변과 실제 문서는 같은 스타일 → 검색 재현율↑

단점: LLM 추가 호출 비용, 가상 답변이 틀리면 검색 방향 왜곡 위험
```

#### Contextual Retrieval (Anthropic, 2024)
각 청크에 **전체 문서 맥락을 요약한 컨텍스트를 prefix로 추가** 후 임베딩.
```
[기존 청크]
"수익률은 3% 증가했습니다."

[Contextual 청크]
"이 내용은 ABC사의 2023년 연간 보고서 중 재무 성과 섹션입니다.
 수익률은 3% 증가했습니다."
→ 임베딩 품질 대폭 향상, 검색 정확도 ~49% 개선 (Anthropic 발표)
```
```python
# 컨텍스트 생성 (프롬프트 캐싱으로 비용 절감)
context_prompt = f"""
<document>{full_document}</document>
다음 청크가 문서 전체에서 어떤 위치/맥락에 있는지 2-3문장으로 설명하라:
<chunk>{chunk}</chunk>
"""
context = llm.call(context_prompt)  # 캐싱 적용 시 청크당 ~1센트
contextual_chunk = f"{context}\n\n{chunk}"
```

#### GraphRAG (Microsoft)
문서를 **지식 그래프로 구조화**해 엔티티 간 관계를 검색에 활용 [Edge et al., 2024].
```
일반 RAG:  청크 단위 유사도 검색 → "ABC사의 CEO는?"에 직접 답변 불가능
GraphRAG:  엔티티(ABC사, 홍길동) + 관계(CEO) 추출 → 그래프 탐색
           → "ABC사 CEO의 이전 직장" 같은 멀티홉 추론 가능
```
- 구축 비용 높음 (LLM으로 엔티티/관계 추출 필요)
- 글로벌 요약 질문에 강함 ("이 보고서의 주요 주제는?")
- 로컬 사실 검색보다 글로벌 이해가 필요한 경우 적합

#### top-k 선택 기준
```
top-k가 너무 작으면: 필요한 청크를 놓침 (recall↓)
top-k가 너무 크면:  노이즈 청크가 LLM을 혼란 (precision↓)

권장 전략:
  1. 초기 검색: top-20~50 (recall 우선)
  2. 리랭킹: cross-encoder로 top-5~10 선별 (precision 복구)
  3. 컨텍스트 윈도우 고려: 토큰 예산 내 최대한 포함
```

---

## 5. 고급 실패 모드 & 복구

> RAG는 **요란하게 죽지 않는다.** 인덱싱 파이프라인은 200을 반환하고, LLM은 매끄러운 한국어 문장을 내고, 데모는 잘 돈다 — 그런데 답이 틀렸다. 아래 실패 모드들은 모두 "에러 없이 잘못된 답"을 만든다는 공통점이 있다. 그래서 **상시 측정**이 유일한 방어선이다 → [[Evaluation]].

### 5.1. 조용한 실패 (silent failure) ⭐
가장 위험한 실패. **답은 그럴듯한데 검색된 컨텍스트가 틀렸다.** 사용자는 자신이 속았다는 것조차 모른다.

```
[정상]   질문 → 올바른 청크 검색 → 근거 기반 답변          → 정답
[조용한 실패] 질문 → 무관한/유사하나 틀린 청크 → 그럴듯한 답변 → 오답인데 자신감 100%
                                                            ↑ 에러 로그 없음, HTTP 200, 매끄러운 문장
```

- **왜 탐지가 어려운가**: 시스템 어디에도 예외가 안 뜬다. 검색은 "top-k 청크"를 정상 반환하고, LLM은 그 청크가 적절한지 판단하지 않고 그냥 쓴다.
- **방어선**:
  - **RAGAS faithfulness**: 답변의 각 주장이 검색된 컨텍스트에 실제로 근거하는가(컨텍스트에 없는 말을 지어냈는가) → 환각 탐지.
  - **RAGAS context precision / recall**: 검색된 청크가 질문에 정말 관련 있는가, 필요한 청크를 다 가져왔는가 → 검색 품질 탐지.
  - **근거 인용 강제**: 답변마다 출처 청크를 함께 출력 → 사람이/평가기가 검증 가능. 인용 못 하면 답하지 말 것.
  - **"모르면 모른다"**: 컨텍스트에 근거가 없으면 답을 만들지 말고 abstain. 시스템 프롬프트에 강제(§5 Spring 예시 system 메시지 참고).
- **운영 원칙**: faithfulness·context precision를 **배포 전 게이트이자 프로덕션 상시 지표**로 둔다. 점수가 떨어지면 데이터/임베딩이 드리프트했다는 신호.

### 5.2. 의미 불일치 연쇄 (vocabulary mismatch)
"X를 검색했는데 사용자가 실제로 필요한 건 Y" — **질문의 표현이 답 문서의 표현을 닮지 않아서** 벡터 공간에서 엉뚱한 곳에 떨어진다.

```
사용자 질문:  "결제가 자꾸 튕겨요"
정답 문서:    "PG사 타임아웃(504) 시 재시도 정책 및 멱등키 처리"
              ↑ 어휘가 전혀 겹치지 않음 → 임베딩 거리 멀어짐 → 검색 실패
```

- **원인**: 질문은 답을 안 닮는다. 사용자는 증상을, 문서는 원인/해법을 다른 어휘로 기술.
- **완화**:
  - **HyDE**(§4): LLM이 "가상 답변"을 만들어 그걸 임베딩 → 답 문서의 어휘에 가까워짐.
  - **query rewriting**: 검색 전에 LLM으로 질문을 재작성/확장(동의어·전문용어 추가, 멀티 쿼리 생성 후 결과 합집합).
  - 하이브리드 검색(§4)으로 키워드 매칭(BM25)을 함께 태워 어휘 격차를 부분 보완.

### 5.3. 임베딩 드리프트 (embedding drift)
**임베딩 모델을 바꾸거나 버전업하면 새 벡터와 기존 인덱스의 벡터가 같은 공간에 있지 않다.** 거리 계산이 무의미해진다.

```
인덱싱 시점:  text-embedding-v1 으로 100만 청크 색인
모델 업데이트: text-embedding-v2 로 질의 임베딩
              → v2 질의 벡터를 v1 인덱스에서 검색 → 좌표계가 다름 → 쓰레기 검색
해결: 전체 재색인(re-index) 필수. 부분 갱신 불가.
```

- **전략**:
  - **모델 버전 고정(pinning)**: 임베딩 모델 ID/버전을 메타데이터에 박아두고, 인덱스 단위로 버전을 추적.
  - **블루-그린 재색인**: 새 모델로 신규 인덱스를 빌드 → 평가 통과 후 스위치. 다운타임/혼합 검색 방지.
  - 재색인 비용(임베딩 API 호출 × 청크 수)을 미리 산정 — 대규모에선 수시간~수일 + 비용. 모델 교체는 가볍게 결정하지 말 것 → [[Embedding]].

### 5.4. 콜드스타트 데이터 품질
사용자가 고통받기 **전에** 인덱스 자체의 병을 진단해야 한다. 나쁜 데이터는 조용한 실패의 1순위 원인.

- **진단 항목**:
  - **중복 청크**: 같은/거의 같은 청크가 top-k를 점령 → 다양성 상실. near-duplicate 탐지(코사인 > 0.95).
  - **임베딩 분포 이상**: 한 점에 뭉친 클러스터(임베딩 실패·빈 청크), 이상치(파싱 깨진 텍스트). 벡터 분포 시각화(UMAP/t-SNE)·노름(norm) 분포 점검.
  - **노이즈 청크**: 머리말/꼬리말/네비게이션 반복, OCR 깨짐, 표가 1차원으로 뭉개진 청크.
  - **빈/초단 청크**: 토큰 수 0~몇 개짜리 → 의미 없는 검색 노이즈.
- **원칙**: "garbage in → silent garbage out". 인덱싱 직후 자동 품질 리포트(중복률·청크 길이 분포·임베딩 노름 분포)를 뽑아 게이트.

### 5.5. 다국어 품질 저하
영어로 평가하면 멀쩡한데 **한국어 검색 품질이 무너지는** 흔한 함정.

- **원인**: 다수 임베딩 모델이 영어 중심으로 학습 → 한국어/비영어 의미 표현이 빈약, 형태소·조사 변형에 약함.
- **완화**:
  - **다국어 임베딩 선택**: `bge-m3`(다국어 + dense/sparse/multi-vector 동시 지원), `multilingual-e5`, Cohere `embed-multilingual` 등.
  - 한국어 코퍼스로 **자체 평가셋**을 만들어 모델별 recall/precision 비교(영어 벤치마크 점수 믿지 말 것).
  - 하이브리드 검색에서 한국어 형태소 분석기(BM25 쪽) 적용 여부도 품질에 직결.

### 5.6. 모니터링 전략
RAG는 조용히 실패하므로 **"에러율"이 아니라 "검색/답변 품질"을 상시 추적**해야 한다.

| 지표 | 의미 | 신호 |
|------|------|------|
| **faithfulness** (RAGAS) | 답이 컨텍스트에 근거하는가 | 하락 → 환각·드리프트 |
| **context precision/recall** | 검색 청크가 적절·충분한가 | 하락 → 청킹/임베딩 문제 |
| **재질문율(re-query)** | 사용자가 곧바로 다시 묻는 비율 | 상승 → 답이 부적절 |
| **인용 클릭/거부율** | 제시한 출처를 보는가 / 답을 버리는가 | 신뢰도 프록시 |
| **abstain율** | "모른다"로 빠진 비율 | 너무 높으면 recall↓, 너무 낮으면 환각 의심 |

- 오프라인 평가셋(골든 Q&A)으로 **회귀 테스트**(인덱스·프롬프트·모델 변경 시 점수 비교).
- 프로덕션 로그에서 질문·검색청크·답변·인용을 샘플링해 LLM-as-judge로 상시 채점.
- 품질 지표는 [[RAG-Latency-Reality]]의 지연 지표와 함께 봐야 한다 — 정확도를 위해 리랭킹·HyDE·재작성을 붙이면 지연이 늘기 때문(정확도↔지연 트레이드오프).

> 핵심: RAG의 SLA는 "200 OK 비율"이 아니라 **"틀린 답을 자신 있게 내놓지 않는 비율"**이다.

---

## 6. Spring 구현 스케치 (Spring AI)
```groovy
implementation 'org.springframework.ai:spring-ai-bom:...'        // VectorStore, EmbeddingModel 추상화
implementation 'org.springframework.ai:spring-ai-pgvector-store'
```
```java
@Service
@RequiredArgsConstructor
public class RagService {
    private final VectorStore vectorStore;     // pgvector 등
    private final ChatClient chatClient;        // LLM 래퍼

    // 인덱싱
    public void index(List<Document> docs) {
        var chunks = new TokenTextSplitter().apply(docs);   // 청킹
        vectorStore.add(chunks);                            // 임베딩+저장 (내부 처리)
    }

    // 질의
    public String ask(String question) {
        List<Document> hits = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(5));     // 벡터 검색
        String context = hits.stream().map(Document::getContent)
                             .collect(Collectors.joining("\n---\n"));
        return chatClient.prompt()
            .system("아래 컨텍스트에 근거해서만 답하고, 모르면 모른다고 답하라. 출처를 표기하라.")
            .user(u -> u.text("질문: {q}\n\n컨텍스트:\n{c}").param("q", question).param("c", context))
            .call().content();
    }
}
```
> 직접 구현 시: 임베딩 호출 → pgvector `<=>` 코사인 거리 정렬 → 프롬프트 합성. [[Redis]]로 임베딩/응답 캐시.

---

## 7. RAG vs 대안
| 방법 | 언제 |
|------|------|
| **RAG** | 자주 바뀌는/방대한/사내 지식, 출처 필요 |
| **긴 컨텍스트에 통째로** | 문서가 작고 적을 때(1M 컨텍스트면 단순) |
| **파인튜닝** | 형식·스타일·말투 고정이 필요, 지식 주입엔 비효율 |
| **툴/함수 호출** | 실시간/구조화 데이터(DB·API) → [[Agent-Harness]]·[[Structured-Data-RAG]](text-to-SQL) |

> 실무에선 **RAG + 툴호출**을 섞는다(검색은 RAG, 실시간 수치는 API 툴).

## 8. 흔한 함정
- 나쁜 청킹 → 검색 품질 전부 무너짐(가장 흔한 원인).
- 질의/문서 임베딩 모델 불일치.
- top-k만 키우기(노이즈↑) → 리랭킹으로 해결.
- 근거 없는 답을 막지 못함 → "컨텍스트에 없으면 모른다고" + 인용 강제.
- 평가 부재 → 검색 재현율/정답률을 **측정**하고 튜닝.

## 9. 관련
- [[LLM]] · [[Agent-Harness]] · [[Prompt-Engineering]] · [[Redis]] · [[RDS-vs-Self-Hosted]] · [[Embedding]] · [[Evaluation]] · [[RAG-Latency-Reality]]
- [[GraphRAG]] — 전역·관계 질문에 강한 그래프 기반 RAG(벡터 RAG의 보완)
- [[Chunking-Strategies]] — 청킹 전략·Contextual Retrieval·late chunking(검색 품질 최대 레버)
- [[Reranking-Retrieval]] — 2단계 검색·cross-encoder 리랭킹·RRF 융합(청킹 다음 ROI 레버)
- [[Query-Transformation]] — 검색 전 쿼리 개선(HyDE·multi-query·step-back·분해)
- [[RAG-Security]] — 간접 인젝션·코퍼스 포이즈닝·데이터 유출(검색 콘텐츠=공격 표면)
- [[Obsidian-RAG-System]] — 이 볼트를 one-shot RAG로 구현한 실사례
- [[Multimodal-RAG]] — 표·차트·PDF 시각 문서 검색(ColPali류)
- [[Agentic-RAG]] — 검색 1회로 안 되는 질문에 자기판단·재검색·보정(Self-RAG·CRAG)
- [[Citation-Attribution]] — 답↔근거 인용 표시·검증(인용 환각·사후 검증)

---

## 참고문헌

[1] P. Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks," *NeurIPS*, 2020. arXiv:2005.11401
[2] L. Gao et al., "Precise Zero-Shot Dense Retrieval without Relevance Labels," *ACL*, 2023. arXiv:2212.10496
[3] S. Robertson & H. Zaragoza, "The Probabilistic Relevance Framework: BM25 and Beyond," *Foundations and Trends in Information Retrieval*, vol. 3, pp. 333–389, 2009.
[4] D. Edge et al., "From Local to Global: A Graph RAG Approach to Query-Focused Summarization," 2024. arXiv:2404.16130
