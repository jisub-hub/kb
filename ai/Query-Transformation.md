---
tags:
  - ai
  - rag
  - query-transformation
  - retrieval
created: 2026-06-19
---

# 쿼리 변환 (HyDE · Multi-query · Step-back · Decomposition)

> [!summary] 한 줄 요약
> 검색 **전에 쿼리를 LLM으로 다듬어** Recall을 올리는 레버. 사용자의 원 질문은 짧고·모호하고·문서 어휘와 다르다 → 변환으로 질문-문서 간극을 메운다. RAG 3단 레버의 **입력측**(인덱싱=청킹, 검색후=리랭킹과 짝).

> 인덱싱은 [[Chunking-Strategies]], 검색 후 재정렬은 [[Reranking-Retrieval]], 파이프라인은 [[RAG]]. 이 노트는 **검색 직전 "쿼리 개선"**.

---

## 1. 왜 — 원 질문은 검색에 불리하다

```
사용자 질문: 짧고·구어체·대명사·문서와 다른 어휘
  "그거 왜 느려?"  ↔  문서: "RAG 파이프라인의 지연은 LLM 생성 단계가 지배한다"
→ 임베딩 유사도·키워드가 어긋남 → 정답 문서를 못 뽑음(Recall 손실)
해법: 검색에 유리한 형태로 쿼리를 변환(LLM 1회 호출 비용으로 Recall↑)
```

## 2. 기법

### 2.1 HyDE (Hypothetical Document Embeddings)
```
쿼리 → LLM이 "가상의 답변/문서"를 생성 → 그 답변을 임베딩해 검색
직관: 질문보다 "답변"이 실제 문서와 어휘·형식이 비슷함 → 매칭↑
주의: LLM이 환각한 가상답변이라도 "임베딩 방향"만 맞으면 됨(내용 정확성 불필요)
      단 도메인이 생소하면 가상답변이 빗나가 역효과 가능
```

### 2.2 Multi-query (쿼리 확장)
```
쿼리 → LLM이 여러 다른 표현/관점으로 N개 생성 → 각각 검색 → 결과 합집합(또는 RRF)
직관: 한 표현이 놓친 문서를 다른 표현이 잡음 → Recall↑
비용: 검색 N배 + 결과 융합(→ [[Reranking-Retrieval]] RRF)
```

### 2.3 Step-back (추상화)
```
구체 질문 → 한 단계 추상화한 "배경 질문"으로 → 원리·배경 지식 검색
예: "X 버전 3.2에서 Y 설정은?" → "X의 Y 설정 일반 원리는?"
직관: 구체 디테일이 문서에 없을 때 상위 개념으로 근거 확보 → 추론 보강
```

### 2.4 Decomposition (분해, multi-hop)
```
복합 질문 → 하위 질문들로 분해 → 각각 검색·답 → 종합
예: "A사와 B사 중 2023 매출 큰 곳은?" → ①A 2023 매출 ②B 2023 매출 → 비교
직관: 단일 검색으로 못 푸는 multi-hop을 단계로 분해 → [[GraphRAG]] 관계 추론과 통함
```

### 2.5 쿼리 라우팅
```
질문 유형을 분류 → 다른 인덱스/도구/검색전략으로 보냄
예: 코드 질문 → 코드 인덱스, 최신 사실 → 웹검색, 일반 → 벡터 RAG
→ LLM이 라우팅 결정하면 [[Agent-Harness]] Agentic RAG로 수렴
```

## 3. 트레이드오프 ⚠️

```
이득: Recall↑·어휘 간극 해소·multi-hop 가능
비용: 검색 전 LLM 호출 추가 → 지연·비용↑(HyDE 1회, multi-query N회, decomposition 다회)
      [[RAG-Latency-Reality]] — 어차피 병목은 최종 생성이라 변환 1~2회는 감내 가능
위험: 변환이 빗나가면(HyDE 환각·잘못된 분해) 오히려 Recall↓ → 도메인서 측정 필수
원칙: 단순 질문엔 변환 생략, 모호·복합 질문에 선택 적용(또는 라우팅으로 분기)
```

## 4. RAG 3단 레버에서의 위치

```
입력측(이 노트)  : 쿼리 변환 — 무엇을 검색할지 다듬기
인덱싱           : 청킹·임베딩 — 무엇을 검색 단위로([[Chunking-Strategies]])
검색 후           : 리랭킹·RRF — 뽑힌 것 재정렬([[Reranking-Retrieval]])
→ 셋은 독립적으로 더해진다. 보통 리랭킹부터 → 청킹 → 쿼리 변환 순으로 ROI 검토.
능동화: LLM이 변환·라우팅·재검색을 스스로 결정 = Agentic RAG([[Agent-Harness]])
```

---

## 5. 관련
- [[RAG]] — 검색 파이프라인(§4 품질 기법)
- [[Reranking-Retrieval]] — 검색 후 재정렬(multi-query 결과 RRF 융합)
- [[Chunking-Strategies]] — 인덱싱측 레버(쿼리 변환과 독립적 보완)
- [[GraphRAG]] — decomposition·multi-hop 관계 추론과 통함
- [[Agentic-RAG]] — LLM이 쿼리 전략·재검색을 능동 결정(이 노트의 변환을 루프로)
- [[Agent-Harness]] — 루프·도구호출·가드레일
- [[Evaluation]] — 변환 전후 Recall@k로 효과·역효과 측정
- [[RAG-Failure-Debugging]] — 어휘 불일치 recall(①) 실패의 수정 위치(HyDE·multi-query)
- [[Conversational-RAG]] — 단독 질의 변환(여기) ↔ 대화 이력으로 후속 질의 standalone 재작성
- [[Self-Query-Retrieval]] — 의미 재작성(여기) ↔ 질의 속 정형 조건을 메타 필터로 분리
