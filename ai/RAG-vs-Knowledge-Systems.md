---
tags:
  - ai
  - rag
  - knowledge-base
  - llm
  - architecture
  - deep-dive
created: 2026-06-16
---

# "LLM Wiki도 결국 RAG 아닌가?" — 전문가 관점 심층 분석

> [!insight] 핵심 질문
> "LLM wiki도 vector embedding을 쓰면 RAG 아닌가?"
> 
> **짧은 답**: 메커니즘은 같다. 그러나 이 질문이 진짜 흥미로운 이유는 — 그 관찰이 옳기 때문에 업계가 RAG를 재정의하고 있기 때문이다.

---

## 1. 지식(Knowledge)을 LLM에 주입하는 방법 — 전체 지도

먼저 LLM에 지식을 넣는 방법을 전부 나열해야 이 논쟁이 정확히 어디서 생기는지 보인다.

```
[지식 주입 방법 분류]

① 파라메트릭 지식 (Parametric Knowledge)
   → 지식이 모델 가중치(weight)에 인코딩
   → 학습 시점에 고정
   → 방법: Pre-training, Fine-tuning, RLHF

② 비파라메트릭 지식 (Non-Parametric Knowledge)
   → 지식이 외부에 있고 추론 시점에 주입
   → 실시간 갱신 가능
   → 방법:
     a) In-Context Stuffing   — 통째로 프롬프트에 넣기
     b) RAG (벡터 검색 기반)  — 필요한 것만 검색해서 넣기
     c) Tool Use / API 호출   — LLM이 직접 조회
     d) Cache-Augmented Gen  — KV Cache에 사전 계산 저장

③ 구조적 지식 (Structured Knowledge)
   → 엔티티·관계를 명시적으로 모델링
   → 방법: Knowledge Graph + RAG (GraphRAG)
```

"LLM Wiki = RAG"라는 관찰은 **② vs ②** 비교다. 즉 비파라메트릭 방법들 안에서 "어떻게 주입하느냐"의 차이 논쟁이다.

---

## 2. RAG란 정확히 무엇인가

Lewis et al. 2020 원논문 정의:

> "RAG combines a *retriever* (dense vector search) with a *generator* (seq2seq) in an end-to-end differentiable system."

핵심은 3가지다:

```
[RAG의 구성 요소]

Retriever
  └─ 쿼리 → 임베딩 → 벡터 DB 유사도 검색 → 관련 문서 k개 반환

Augment
  └─ 검색된 문서를 프롬프트 컨텍스트에 추가

Generator
  └─ [시스템 프롬프트 + 검색 결과 + 사용자 질문] → LLM → 답변
```

**이 정의에 따르면** 벡터 임베딩으로 wiki를 검색해서 LLM에 넣는 것은 **RAG가 맞다.** 사용자의 관찰이 기술적으로 정확하다.

---

## 3. 그런데 왜 업계에서 "RAG"와 "LLM Wiki/KB"를 다르게 부르는가

두 가지 이유가 있다:

### 이유 1 — RAG는 플랫한 청크 검색, Wiki는 구조를 가진다

```
[Naive RAG의 세계관]
  모든 문서 = 500 토큰짜리 청크의 나열
  검색 = 코사인 유사도 Top-K
  
  결과: "HNSW 인덱스" 쿼리 →
    [청크 A: HNSW 파라미터 설명]
    [청크 B: pgvector 설치 방법]
    [청크 C: IVFFlat vs HNSW 비교]
  → LLM이 이 3개 청크로 답변 생성

[구조화된 Wiki의 세계관]
  문서 = 계층 구조 + 명시적 링크 + 메타데이터
  
  pgvector.md
    └─ 링크: [[PostgreSQL-Internals]], [[RAG]], [[DB-Fundamentals]]
    └─ 태그: database, vector, index
    └─ 섹션: 개념 / 설치 / 인덱스 / 성능튜닝 / Spring연동
    
  검색 = 유사도 + 그래프 링크 순회 + 섹션 구조 파악
```

순수 RAG는 **청크 A, B, C가 같은 문서의 다른 섹션인지, 전혀 다른 맥락인지 모른다.**
Wiki 구조에서는 이것이 명시적으로 표현된다.

이 차이가 **Microsoft GraphRAG** 탄생의 동기다.

### 이유 2 — 언제 검색할지 (When to retrieve)

```
[Traditional RAG]
  모든 질문 → 무조건 검색 → 결과를 컨텍스트에 추가 → 생성
  = 검색이 강제되는 파이프라인

[LLM Wiki / Agentic 접근]
  질문 → LLM이 판단: "내가 아는가? 검색이 필요한가?"
        → 필요하면 어느 문서를 볼지 선택 (Tool Call)
        → 문서 읽기 → 추가 탐색 여부 판단 (recursion)
        → 답변 생성
```

후자는 LLM이 **능동적으로 지식 베이스를 탐색**한다. RAG는 수동적 검색 파이프라인이다.

이 차이가 **MemGPT / OpenAI Memory / Agentic RAG**가 등장한 배경이다.

---

## 4. 같은 벡터 임베딩인데 왜 결과가 다른가 — 실제 사례

```python
# 케이스: "K8s에서 DB를 Pod로 올리면 안 되는 이유가 뭐야?"

# ── Naive RAG ──────────────────────────────────────────────────
쿼리 임베딩 → Top-3 청크:
  1. StatefulSet-DB.md (코사인 0.87): "Pod는 일회성이며 상태를 갖지 않는다..."
  2. PV-PVC.md (코사인 0.81): "PersistentVolume은 노드에 바인딩된다..."
  3. K8s-Fundamentals.md (코사인 0.79): "Kubernetes는 선언적 배포를..."

→ LLM은 이 3개 청크만으로 답변 생성
→ "StatefulSet-DB.md가 PostgreSQL-HA-Cluster.md를 링크하고 있다"는 정보는 손실
→ "그래서 어떻게 해야 하냐"에 대한 Patroni/CNPG 대안은 안 나옴

# ── 구조 인식 검색 ─────────────────────────────────────────────
쿼리 임베딩 → StatefulSet-DB.md 발견
  → 해당 문서의 링크 탐색:
      [[PostgreSQL-HA-Cluster]] (연관 0.8)
      [[DB-Deployment-Decision]] (연관 0.75)
  → 추가 청크 수집

→ LLM은 "K8s Pod 문제점 + Patroni 대안 + K8s Operator(CNPG) 대안" 전부 답변
→ 실제로 더 유용한 답변
```

**벡터 임베딩은 동일하게 썼지만 결과가 다르다.** 차이는 구조를 활용하느냐 아니냐에 있다.

---

## 5. RAG의 진화 — 왜 계속 새 이름이 붙는가

이 질문("다 RAG 아닌가")이 계속 제기되기 때문에 업계는 RAG를 세분화했다.

```
[RAG 진화 계보]

Naive RAG (2020~2022)
  ├─ Dense retrieval (DPR)
  ├─ Chunking → 임베딩 → Top-K → 컨텍스트 삽입
  └─ 문제: 청크 분절, 관련성 낮은 청크 혼입, 구조 손실

Advanced RAG (2022~2023)
  ├─ Query Expansion / HyDE (가상 답변으로 쿼리 보강)
  ├─ Re-ranking (Cross-Encoder로 Top-K 재정렬)
  ├─ Hybrid Search (BM25 + 벡터 RRF 결합)
  ├─ Contextual Retrieval (Anthropic 2024: 청크에 문서 맥락 주입)
  └─ Parent-Child Chunking (작은 청크 검색 → 큰 부모 청크 반환)

Modular RAG (2023~)
  ├─ Routing: 쿼리 유형에 따라 다른 Retriever 선택
  ├─ Fusion: 여러 검색 결과 병합·재랭킹
  ├─ Step-back Prompting: 추상적 질문으로 먼저 검색
  └─ FLARE: 생성 중 불확실 구간에서 추가 검색 트리거

GraphRAG (Microsoft 2024)
  ├─ 엔티티 추출 → Knowledge Graph 구축
  ├─ 커뮤니티 탐지 → 요약 계층 생성
  ├─ Local search: 특정 엔티티 관련 쿼리
  ├─ Global search: 전체 데이터셋 요약 쿼리
  └─ 핵심: "이 토픽과 관련된 모든 엔티티와 관계는?"

Agentic RAG (2024~현재)
  ├─ LLM이 검색 전략을 스스로 결정
  ├─ Multi-hop: A → B → C 연쇄 검색
  ├─ Self-correction: 검색 결과 불충분 시 재검색
  └─ Tool-augmented: 검색 외 계산·API 호출 병행
```

이 진화 전체가 "더 구조적으로, 더 능동적으로"를 향한다. 즉 **좋은 LLM Wiki가 하는 것**에 수렴한다.

---

## 6. 그렇다면 "LLM Wiki"는 어디에 위치하는가

```
[스펙트럼]

──────────────────────────────────────────────────────────────────
비구조적                                               구조적
플랫 검색 <──────────────────────────────────────────> 그래프 탐색
──────────────────────────────────────────────────────────────────
Naive RAG          Advanced RAG       GraphRAG       LLM Wiki
(청크 Top-K)       (하이브리드+리랭킹)  (엔티티+관계)  (명시적 구조+링크+
                                                     계층+버전+편집권)

──────────────────────────────────────────────────────────────────
수동적 파이프라인 <─────────────────────────────────────> 능동적 탐색
──────────────────────────────────────────────────────────────────
Naive RAG                                            Agentic RAG
(쿼리→검색→생성)                                     (판단→탐색→검색→
                                                      추가탐색→생성)
```

**LLM Wiki는 RAG의 오른쪽 끝에 있는 최적화된 형태다.** RAG의 부정이 아니라 RAG의 완성에 가깝다.

---

## 7. Long Context 모델이 RAG 자체를 무너뜨리는 시나리오

2024~2026년에 등장한 진짜 위협: **컨텍스트 창이 너무 커져서 검색이 불필요해지는 경우.**

```python
# 1M context 모델 (Claude, Gemini 1.5) 시나리오

# 소규모 wiki (200개 파일, ~40K 라인 = ~약 500K 토큰):
# → 전부 컨텍스트에 넣어도 됨
# → 검색 불필요
# → 이건 RAG가 아니라 "CAG (Cache-Augmented Generation)" 또는
#    그냥 "long-context prompting"

# 대규모 wiki (10만 문서, 수억 토큰):
# → 컨텍스트에 다 못 넣음
# → 검색 필수 → RAG 여전히 필요
```

현재 이 Obsidian vault (~213 파일, ~40K 라인, ~50만 토큰):

```python
# 이 vault를 전부 Claude에게 줄 수 있는가?
vault_tokens = 500_000  # 추정
claude_context = 200_000  # claude-sonnet-4-6 기준

# 200K 컨텍스트로는 전부 못 넣음 → 검색 필요 → RAG 유효
# 만약 claude-opus-4-8 1M 컨텍스트라면 → 전부 넣기 가능 → RAG 불필요
```

**결론**: Long-context가 확장될수록 소규모 지식베이스에서는 RAG가 불필요해진다. "RAG는 컨텍스트 창의 물리적 한계를 우회하는 기술"이기도 하다.

---

## 8. 검색 없는 지식 접근 — CAG vs RAG 비교

```
[CAG: Cache-Augmented Generation]
  ─────────────────────────────────────────────────────────
  아이디어: KV Cache는 계산 비용이 싸다.
           문서 전체를 미리 KV Cache로 만들어두면
           토큰을 다시 처리하지 않아도 된다.
  
  흐름:
    1. 전체 지식베이스 → LLM KV Cache 사전 계산 (한 번)
    2. 쿼리 → KV Cache에 어텐션 → 답변
    
  장점:
    - 검색 단계 없음 (Retrieval 오류 없음)
    - 전체 문서에 어텐션 가능
    - 청킹 분절 문제 없음
    
  단점:
    - KV Cache 크기 = 문서 크기에 비례 (메모리 폭발)
    - 문서 갱신 시 전체 재계산 필요
    - 수십만 문서 규모에서 비현실적

[RAG]
  장점:
    - 문서 규모 무제한 (검색이 필터링)
    - 실시간 추가 가능
    - 관련 문서만 컨텍스트 → 집중도 높음
    
  단점:
    - Retrieval 실패 → 생성 실패
    - 청크 분절로 맥락 손실
    - 검색 지연 (수십~수백ms)
```

**현실적 결론**: CAG는 소규모 고정 지식에, RAG는 대규모 동적 지식에 적합하다. 둘 다 "벡터 임베딩 없이도 작동 가능"하다는 점에서 RAG와 LLM Wiki의 구분은 더 복잡해진다.

---

## 9. Knowledge Graph RAG — 구조적 지식이 왜 중요한가

Microsoft GraphRAG 논문(2024)이 제기한 핵심 주장:

> "Naive RAG는 'What is X?' 질문에는 잘 답하지만, 'What are the major themes across this entire corpus?' 같은 global query에는 실패한다."

```python
# Naive RAG로 답하기 어려운 질문들:
질문들 = [
    "이 프로젝트의 전체 기술 스택에서 가장 복잡한 부분은?",
    "Spring과 Infra 섹션에서 공통적으로 나오는 HA 패턴은?",
    "이 KB에서 가장 많이 연결된 핵심 개념 5개는?",
]

# → 이 질문들은 특정 청크를 검색하는 게 아니라
# → 전체 문서 간 관계를 이해해야 답 가능
# → Knowledge Graph가 이를 가능하게 함

# GraphRAG 흐름:
# 1. 모든 문서에서 엔티티 추출 (PostgreSQL, Patroni, etcd, HA...)
# 2. 관계 추출 (Patroni --manages--> PostgreSQL, etcd --coordinates--> Patroni)
# 3. 커뮤니티 탐지 (HA Cluster 클러스터, RAG 파이프라인 클러스터...)
# 4. 커뮤니티별 요약 생성
# 5. 글로벌 쿼리 → 커뮤니티 요약 검색 → 종합 답변
```

**이 Obsidian vault의 `[[wikilink]]` 구조는 수동으로 만든 Knowledge Graph다.** 자동 GraphRAG보다 정확도가 높다. 왜냐면 사람이 "이 문서는 저 문서와 관련있다"고 명시적으로 판단했기 때문이다.

---

## 10. 실용적 결론 — 무엇을 써야 하는가

```
[상황별 선택]

소규모 고정 지식 (<500K 토큰):
  → Long-context 모델에 전부 넣기 (CAG)
  → 검색 불필요, 구현 단순
  → 단점: 비용 (매 쿼리마다 모든 토큰 처리)

중규모 동적 지식 (수천~수만 문서):
  → Advanced RAG + 구조 인식
  → Hybrid search (BM25 + 벡터) + Re-ranking
  → 가능하면 위키링크(wikilink) 같은 명시적 링크 활용
  → 이것이 "LLM Wiki" 접근

대규모 비정형 지식 (수십만~수백만 문서):
  → GraphRAG (엔티티·관계 자동 추출)
  → 글로벌 쿼리에 강함
  → 구축 비용 높음

도메인 특화 지식 (정확도 극단적으로 중요):
  → Fine-tuning (파라메트릭 지식화)
  → + RAG 병행 (최신 정보 보완)
```

---

## 11. 최종 답변 — "LLM Wiki는 RAG인가"

```
"LLM Wiki가 vector embedding을 쓰면 RAG 메커니즘을 사용하는 것은 맞다.
 그러나 '다 RAG다'는 관찰은 절반만 옳다."

옳은 부분:
  ✅ Retrieve → Augment → Generate 구조는 동일
  ✅ 벡터 유사도 검색은 같은 수학
  ✅ 청크를 컨텍스트에 넣는 것도 같은 메커니즘

놓치는 부분:
  ❌ RAG는 플랫 청크 검색, 좋은 LLM Wiki는 구조적 관계까지 활용
  ❌ Naive RAG는 수동적 파이프라인, Agentic 시스템은 능동적 탐색
  ❌ Long context가 확장되면 "검색" 단계 자체가 불필요해질 수 있음
  ❌ Knowledge Graph 기반 접근은 RAG의 다른 패러다임

진짜 통찰:
  "LLM Wiki를 잘 만들면 Naive RAG보다 훨씬 좋은 RAG가 된다."
  
  이 Obsidian vault의 위키링크(wikilink) 구조 + 태그 + MOC 계층은
  수동으로 만든 Knowledge Graph다.
  이것을 벡터 검색과 결합하면:
  
  Best RAG = 벡터 유사도 검색 (Naive RAG)
           + wikilink 그래프 탐색 (GraphRAG)
           + MOC 계층 구조 (Modular RAG)
           + LLM이 추가 탐색 여부 결정 (Agentic RAG)
```

---

## 11.5 지식 주입 4축 — 한눈 비교 & 선택 ⭐

> 무엇으로 검색하느냐(또는 검색을 하느냐)가 갈림길. **검색 단위**가 본질 차이다.

| 축 | 지식 위치 | 검색 단위 | 강점 | 약점/비용 |
|----|----------|----------|------|----------|
| **LLM Wiki** | 마크다운 파일(컨텍스트) | 파일 통째(인덱스로 선택) | 단순·정확·읽힘·디버깅 쉬움 | 규모 한계(~수백 페이지·컨텍스트) |
| **Vanilla RAG** | 외부 벡터 DB | 텍스트 청크(유사도) | 범용·저비용·대규모 | 멀티홉·전역 질문 약함 |
| **GraphRAG** | 지식 그래프 + 벡터 | 엔티티·관계·커뮤니티(순회+유사도) | 멀티홉·전역·설명가능 | 인덱싱 비용·갱신·지연 ([[GraphRAG]]) |
| **Parametric** | 모델 가중치(LoRA/모듈) | 검색 없음(내장) | 추론 빠름·컨텍스트 0 | 갱신·삭제 난제·편집 정합 ([[Parametric-Knowledge-Editing]]) |

```
선택 트리:
  지식 작음(<수백 페이지)·자주 고침·디버깅 중요  → LLM Wiki  (= 이 Obsidian vault·[[Agent-Persistent-Memory]])
  지식 큼·관계 단순·단순 Q&A                      → Vanilla RAG ([[RAG]])
  관계 복잡·멀티홉·규제(설명가능성) 필요          → GraphRAG  (비용 감당 가능할 때만)
  추론 속도·내장 지식·컨텍스트 절약이 핵심        → Parametric (성숙도 낮음, 주시)
하이브리드(실전 수렴): Wiki로 핵심 앵커 + GraphRAG로 관계 + 벡터로 대량 폴백 (+ Agentic 탐색)
+ 5번째 변형: 지식이 컨텍스트에 다 들어가고 안 바뀌면 → CAG(전부 주입+KV 캐시, 검색 0) → [[CAG-Cache-Augmented-Generation]]
```

> [!note] 수치 주의
> "GraphRAG 종합성 +72%·정밀도 +35%·인덱싱 $33K" 같은 벤치/사례 숫자는 **출처(벤더 블로그·특정 데이터셋)별로 크게 다르다** — 방향성 참고용이지 보편 수치 아님. 도메인에서 직접 측정([[Evaluation]]) 후 판단.

---

## 12. 관련
- [[RAG]] — 검색 증강 생성 기술 상세
- [[Embedding]] — 임베딩 모델, 벡터 유사도
- [[Agentic-RAG]] — 능동·반복 검색 패러다임 상세(Self-RAG·CRAG·multi-hop)
- [[Agent-Harness]] — Agentic RAG 구현 (LLM이 검색 전략 결정)
- [[RAG-Chatbot-E2E]] — E2E 구현 예시
- [[Fine-Tuning]] — 파라메트릭 지식화 (RAG의 대안)
- [[Parametric-Knowledge-Editing]] — 파라메트릭 지식을 모듈 단위로 편집·핫스왑(DMoE), RAG와 상보
- [[GraphRAG]] — 그래프 기반 RAG 심화(§9 확장): 엔티티/관계·커뮤니티·global/local 검색·비용
- [[LLM-Wiki]] — "그럼 LLM Wiki를 어떻게 만드나"의 실천 구축 가이드(이 노트는 정의·위치)
- [[LLMOps]] — 어떤 방식이 비용 효율적인가
