---
tags:
  - ai
  - rag
  - agentic-rag
  - retrieval
  - agent
created: 2026-06-22
---

# Agentic RAG — 반복·교정 검색 (Self-RAG · CRAG)

> [!summary] 한 줄 요약
> **"검색 1회 → 생성"이 안 통하는 질문**에 RAG가 스스로 *검색 여부·품질 평가·재검색·보정*을 판단해 루프를 도는 패러다임. 정적 파이프라인([[RAG]])에 **결정·평가·반복**을 얹은 capstone. 입력측 [[Query-Transformation]]·검색후 [[Reranking-Retrieval]]를 LLM이 능동적으로 엮으면 여기로 수렴한다([[Agent-Harness]] 루프).

> 정적 RAG는 [[RAG]], 쿼리 다듬기는 [[Query-Transformation]], 검색 품질 재정렬은 [[Reranking-Retrieval]], 루프 메커니즘은 [[Agent-Harness]]. 이 노트는 **검색 자체를 능동·반복으로** 만드는 축.

---

## 1. 왜 — 정적 파이프라인의 한계

```
정적 RAG: 쿼리 → 검색(1회) → top-k 주입 → 생성   (검색이 빗나가도 그대로 생성→환각)
한계:
  - 검색 결과가 부실해도 "그대로" 답함(품질 게이트 없음)
  - 애초에 검색이 불필요한 질문(상식·요약)도 항상 검색(비용·노이즈)
  - multi-hop("A·B 중 매출 큰 곳?")은 한 번 검색으론 부족
해법: LLM이 루프 안에서 스스로 판단 — "검색할까? 결과 쓸 만한가? 더 검색할까?"
```

이건 [[RAG-vs-Knowledge-Systems]]의 "Agentic RAG" 칸 — 검색을 **모델의 행동(action)**으로 끌어올린 것.

## 2. Self-RAG — reflection 토큰으로 자기 판단

```
모델이 생성 중 특수 토큰(reflection token)을 내며 스스로 결정:
  ① Retrieve?     이 질문/시점에 검색이 필요한가 (No면 파라메트릭 지식으로 답)
  ② IsRelevant?   뽑힌 each 패시지가 질문에 관련 있나 (관련만 사용)
  ③ IsSupported?  생성한 문장이 근거 패시지에 충실한가 (환각이면 폐기·재생성)
  ④ IsUseful?     최종 답이 실제로 유용한가 (자기 채점)
핵심: 검색 여부·근거 충실성을 "학습된 토큰"으로 내재화 → 불필요 검색 줄이고 환각 억제
주의: reflection 토큰을 내도록 파인튜닝된 모델이 전제(범용 모델은 프롬프트로 근사)
```

→ ③ IsSupported는 [[Evaluation]]의 Faithfulness를 **추론 시점에 셀프 게이트**로 넣은 것.

## 3. CRAG (Corrective RAG) — 검색 품질 등급 → 보정

```
검색 → retrieval evaluator(경량 grader)가 결과 품질을 3등급으로:
  Correct    : 충분 → top-k 정제(knowledge refine)해서 그대로 생성
  Incorrect  : 부실 → 폐기하고 **웹 검색**(외부)으로 대체 수집
  Ambiguous  : 애매 → 내부 결과 + 웹 검색 둘 다 결합
직관: "검색이 실패할 수 있다"를 1급 시민으로 — grader가 게이트, 웹검색이 폴백(fallback)
포인트: 평가자는 작고 빠른 모델/분류기. 비싼 LLM 생성 전에 품질을 거름
```

> [!note] Self-RAG vs CRAG
> - **Self-RAG**: 검색 여부·근거 충실성을 **모델 내부**(학습된 토큰)에서 결정.
> - **CRAG**: 검색 품질을 **외부 grader**로 평가하고 부실하면 **웹검색으로 보정**.
> - 직교적이라 결합 가능(셀프 판단 + 외부 폴백).

## 4. 반복·멀티스텝 검색 (iterative / multi-hop)

```
검색 → 추론 → "근거 부족" 판단 → 쿼리 재구성 → 재검색 → ... (충분해질 때까지)
multi-hop 예: "A의 창업자가 다닌 대학의 설립연도?"
  hop1: A 창업자=? → hop2: 그 사람 대학=? → hop3: 그 대학 설립연도=?
관련: 쿼리 분해는 [[Query-Transformation]] decomposition, 관계 추적은 [[GraphRAG]]
차이: 분해(Query-Transformation)는 "미리 쪼갬", 반복검색은 "결과 보고 다음을 정함"(동적)
```

## 5. 에이전트 결정 — 검색을 도구로

```
LLM이 루프에서 행동을 선택([[Agent-Harness]]):
  - search(query)   언제·무엇을 검색할지
  - 어느 인덱스/도구  벡터RAG·웹·코드·DB (= [[Query-Transformation]] 라우팅)
  - 재시도 여부       결과 보고 다시 검색할지·다른 전략 쓸지
  - 종료 판단        충분한 근거가 모였는지
→ ReAct(추론↔행동 교대) 류 루프. "검색"이 thought→action→observation의 한 action.
```

## 6. 트레이드오프 ⚠️

```
이득 : 정확도·근거 충실성↑, 불필요 검색 절감(Self-RAG), 실패 복구(CRAG 폴백), multi-hop 가능
비용 : LLM 호출 다회(판단+재검색+재생성) → 지연·토큰 비용 급증
       → [[RAG-Latency-Reality]] 병목이 생성인데, 루프는 그 생성을 여러 번 돈다(누적)
위험 : ★루프 폭주★ — grader 오판·종료조건 미흡 시 무한 재검색/재생성
대응 : max iterations 한도·시간/토큰 예산·신뢰도 임계로 조기 종료([[Agent-Harness]] 가드레일)
원칙 : 대부분 질문엔 정적 RAG로 충분. **검색 1회로 안 되는** 복합·고위험 질문에만 능동화(질·비용 균형).
```

## 7. RAG 진화 스펙트럼에서의 위치

```
정적 RAG        : 쿼리 → 검색1회 → 생성              (기본·최저비용)
+ 쿼리변환       : 검색 전 다듬기                      ([[Query-Transformation]])
+ 리랭킹         : 검색 후 재정렬·품질↑               ([[Reranking-Retrieval]])
+ Agentic(이 노트): 검색 여부·품질·반복을 LLM이 결정    (능동·고비용·고정확)
→ 앞 레버들을 "조건부·반복"으로 묶은 게 Agentic RAG. 루프는 [[Agent-Harness]].
```

---

## 8. 관련
- [[RAG]] — 정적 검색 파이프라인(이 노트가 능동화하는 기반)
- [[Query-Transformation]] — 쿼리 변환·라우팅(Agentic의 입력측 행동)
- [[Reranking-Retrieval]] — 검색 후 품질 평가(CRAG grader와 통함)
- [[Agent-Harness]] — 루프·도구호출·가드레일(루프 폭주 한도)
- [[RAG-vs-Knowledge-Systems]] — Agentic RAG의 전체 지도상 위치
- [[GraphRAG]] — multi-hop 관계 추론(반복검색과 보완)
- [[Citation-Attribution]] — IsSupported를 파이프라인 밖으로: 인용 표시·사후 검증
- [[Evaluation]] — Faithfulness/Recall(Self-RAG 셀프게이트의 측정 대응)
- [[RAG-Latency-Reality]] — 반복 루프의 지연·비용 누적 현실
