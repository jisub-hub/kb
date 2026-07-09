---
tags:
  - ai
  - rag
  - retrieval
  - metadata-filter
  - self-query
created: 2026-06-22
---

# Self-Query 검색 — NL에서 메타데이터 필터 자동 추출

> [!summary] 한 줄 요약
> "2023년 이후 작성된 **보안** 문서에서 인증 설정"처럼 자연어 질의에는 *조건*(날짜·카테고리·저자)과 *의미*(인증 설정)가 섞여 있다. 조건을 의미 검색(임베딩)에 맡기면 빗나간다 — 숫자·날짜·범주는 벡터 공간에서 가깝지 않기 때문. **self-query**는 LLM으로 질의를 `{메타데이터 필터} + {잔여 의미 질의}`로 분해해, 벡터 스토어의 **구조화 필터(pre-filter)** 와 벡터 검색을 결합한다. 핵심 함정은 **over-filtering→0건**과 필터 추출 오류.

> 의미 질의 재작성은 [[Query-Transformation]](HyDE·multi-query), 별도 DB로의 text-to-SQL은 [[Structured-Data-RAG]], 저장 스키마는 [[KB-Database-Schema]]. 이 노트는 **NL→벡터 스토어 메타 필터 자동 추출**.

---

## 1. 왜 조건을 의미 검색에 맡기면 실패하나

```
질의: "2023년 이후 보안 카테고리 문서에서 인증 설정 방법"
순진한 벡터 검색: 전체를 임베딩 → "2023", "보안 카테고리"가 의미 신호로 희석
  → 2021년 문서·다른 카테고리가 상위에 섞임(조건이 강제되지 않음)
근본: 날짜 범위·정확 범주·저자는 "유사"가 아니라 "참/거짓 필터"여야 함
  → 메타데이터 필드에 대한 구조화 조건 = DB WHERE 절의 일
```

## 2. self-query 분해 ⭐

```
LLM이 질의를 둘로 가른다(스키마를 프롬프트에 제공):
  입력: "2023년 이후 보안 문서에서 인증 설정"
  출력(구조화):
    filter: { and: [ {date: {gte: "2023-01-01"}}, {category: {eq: "security"}} ] }
    query : "인증 설정 방법"          ← 이 잔여 질의만 임베딩·벡터 검색
→ 벡터 스토어에 filter + query를 함께 던짐(메타 필터 AND 의미 유사)
LangChain SelfQueryRetriever, LlamaIndex auto-retrieval이 이 패턴.
```

> [!tip] 스키마 인지가 정확도를 만든다
> LLM에 **허용 필드·타입·연산자·예시 값**을 명시해야 한다("category는 enum: security|billing|...", "date는 ISO"). 자유 추출은 존재하지 않는 필드·잘못된 연산자를 환각 → 실행 오류나 빈 결과. 추출된 필터는 화이트리스트로 검증.

## 3. pre-filter vs post-filter

```
pre-filter (검색 전 메타로 후보 축소):
  - 정확·완전(조건 밖은 절대 안 나옴), 카디널리티 높은 필드에서 속도도 이득
  - 단, 일부 ANN 인덱스는 강한 pre-filter 시 그래프 탐색이 비효율(필터드 ANN 한계)
post-filter (벡터 top-k 뽑고 나서 메타로 거름):
  - 인덱스 친화적이나, top-k가 죄다 필터에서 탈락하면 결과 부족(recall 손실)
실무: 벡터 스토어의 네이티브 필터(pgvector WHERE·Qdrant payload filter)를 쓰면
      대개 pre-filter로 정확·효율을 같이 잡음([[Embedding]] §5 벡터 DB)
```

## 4. ★ 함정 ★

```
✗ over-filtering → 0건: 필터가 과도/오추출되면 후보가 비어 답을 못 함
   완화: 0건이면 필터 완화·재시도(점진적 relaxation), 사용자에게 필터 노출
✗ 필터 추출 오류: "최근"을 날짜로 못 바꾸거나, 범주명을 오매핑
   완화: enum·동의어 사전 제공, few-shot 예시, 추출 결과 검증
✗ 의미 신호 유실: 조건어를 필터로 빼되 잔여 질의가 너무 빈약해지면 벡터 검색 약화
✗ 권한 필터와 혼동: ACL은 사용자가 못 끄는 강제 필터 — self-query 추출과 분리해
   항상 AND로 강제([[RAG-Security]]·[[KB-Access-Control]]). 추출 필터는 "검색 편의"
✗ 인젝션: NL이 필터 DSL이 되는 경로 — 화이트리스트·파라미터화로 임의 필드 접근 차단
```

## 5. 언제 쓰나

```
쓸 값어치: 문서에 풍부한 정형 메타(날짜·저자·부서·문서종류·상태)가 있고,
          사용자 질의가 자주 그 축으로 좁혀질 때(로그·정책·티켓·논문 코퍼스)
불필요: 메타가 빈약하거나 질의가 순수 의미형이면 오버엔지니어링
조합: self-query(필터) + 하이브리드(BM25+dense) + 리랭킹이 함께 가장 강함
```

## 6. RAG 검색 기법 지도에서의 위치

```
질의 가공 3축:
  의미 재작성:  [[Query-Transformation]] (HyDE·multi-query·step-back) — 무엇을 어떻게 물을까
  대화 해소:    [[Conversational-RAG]] — 이력으로 후속 질의를 자립화
  조건 분리:    이 노트 — 질의 속 정형 조건을 메타 필터로 빼냄
다른 데이터로 가는 경로: [[Structured-Data-RAG]] (text-to-SQL, 별도 DB)
  ↔ self-query는 "같은 벡터 스토어"의 메타 필터(데이터를 옮기지 않음)
```

---

## 7. 관련
- [[RAG]] — §메타데이터 필터링을 NL 자동 추출로 구체화
- [[Query-Transformation]] — 의미 질의 재작성(이 노트는 조건 분리 — 상보)
- [[Structured-Data-RAG]] — 별도 DB로 text-to-SQL(이 노트는 벡터 스토어 내 메타 필터)
- [[Embedding]] — 벡터 DB의 네이티브 메타 필터(pgvector·Qdrant)·pre vs post
- [[KB-Database-Schema]] — 필터 대상이 되는 메타데이터 모델(taxonomy·날짜·권한)
- [[RAG-Security]] — ACL 강제 필터는 추출 필터와 분리해 항상 AND
- [[Conversational-RAG]] — 멀티턴에선 재작성 후 필터 추출(순서)
- [[RAG-Failure-Debugging]] — over-filtering(0건)은 recall(①) 실패의 한 원인
