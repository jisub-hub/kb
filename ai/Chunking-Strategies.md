---
tags:
  - ai
  - rag
  - chunking
  - retrieval
  - embedding
created: 2026-06-19
---

# 청킹 전략 & Contextual Retrieval (RAG 검색 품질의 최대 레버)

> [!summary] 한 줄 요약
> RAG 품질은 **무엇을 어떻게 쪼개 임베딩하느냐**에서 가장 크게 갈린다. 단순 고정 청킹은 **청크가 문서 맥락을 잃어** 검색이 실패한다. 현대 해법은 ① **Contextual Retrieval**(청크에 LLM으로 문서 맥락을 prepend) ② **late chunking**(전체 임베딩 후 청크 풀링) ③ 구조 인식 청킹 + 리랭킹. 청킹은 가장 싸게 큰 효과를 보는 레버다.

> RAG 파이프라인은 [[RAG]], 임베딩 운영은 [[Embedding]], 그래프 검색은 [[GraphRAG]]. 이 노트는 **인덱싱 직전 "쪼개기·맥락 부여"** 심화.

---

## 1. 왜 청킹이 RAG 품질을 지배하나

```
임베딩·검색은 "청크 단위"로 일어난다 → 청크가 나쁘면 그 위 모든 게 나쁨
대표 실패: 청크가 문맥을 잃음
  원문: "...그 회사의 2023년 매출은 전년 대비 3% 증가했다."
  청크만 보면 "그 회사"가 누구? "전년"이 언제? → 쿼리 "ACME 2023 매출"과 매칭 실패
→ 검색이 틀리면 LLM이 아무리 좋아도 답이 틀림(garbage in)
```

## 2. 청킹 전략 스펙트럼

| 전략 | 방식 | 장단 |
|------|------|------|
| **Fixed-size** | N토큰마다 자름(+overlap) | 단순·빠름 / 의미 경계 무시 |
| **Recursive** | 구분자 계층(문단→문장→…)으로 자름 | 구조 일부 존중(기본 권장 출발점) |
| **Semantic** | 임베딩 유사도 급변 지점에서 분할 | 의미 경계 보존 / 계산 비용↑ |
| **Document-structure** | 마크다운 heading·표·코드블록 단위 | 구조 문서에 최적(이 vault 같은) |
| **Parent-document** | 작은 청크로 검색 → 큰 부모를 LLM에 전달 | 검색 정밀 + 맥락 보존 |

```
공통 옵션:
  overlap(겹침) — 경계 잘림 완화(보통 10~20%). 과하면 중복·비용↑
  메타데이터 부착 — 제목·섹션·출처·날짜를 청크에 → 필터·맥락
  표/코드 — 행 단위로 쪼개면 깨짐 → 통째 유지하거나 구조 인식 분할
```

## 3. ⭐ Contextual Retrieval (Anthropic, 2024)

```
아이디어: 청크를 임베딩/색인하기 "전에", LLM으로 그 청크가 문서 어디에 속하는지
          한두 줄 맥락을 생성해 청크 앞에 prepend
  "이 청크는 ACME Corp 2023 연차보고서의 재무 섹션 일부다." + 원래 청크
효과: 청크가 자기 맥락을 갖게 됨 → 검색 실패율 대폭↓
구성: contextual embedding(위 prepend 후 임베딩) + contextual BM25(같은 텍스트로 키워드 색인)
      + 리랭킹(상위 후보 재정렬) → 셋 결합이 가장 강함
```
> [!tip] 비용은 prompt caching으로 잡는다
> 청크마다 "문서 전체 + 그 청크"를 LLM에 넣어 맥락 생성 → 순진하게 하면 비쌈.
> **문서 전체를 프롬프트 캐시에 올려두고 청크만 바꿔** 호출 → 입력 토큰 재사용으로 저렴([[Prompt-Caching]]). 그래도 인덱싱은 1회성 배치 비용이 든다(트레이드오프).

## 4. Late Chunking (Jina, 2024)

```
순서를 뒤집는다:
  일반: 청크로 자름 → 각 청크를 따로 임베딩(청크는 서로를 못 봄)
  late: 긴 컨텍스트 임베딩 모델로 "문서 전체"를 먼저 임베딩(토큰 단위 임베딩 획득)
        → 그 토큰 임베딩을 청크 경계로 평균 풀링 → 청크 임베딩
효과: 각 청크 임베딩이 "전체 문서 맥락을 본 상태" → 대명사·참조 해소
Contextual Retrieval과 차이: CR은 "텍스트를 추가"(LLM 호출),
                            late chunking은 "임베딩 방식을 바꿈"(LLM 호출 없음, 임베딩 모델 의존)
```

## 5. 청크 크기 트레이드오프

```
작은 청크: 검색 정밀(쿼리와 딱 맞는 조각) / 맥락 부족·답 조각남
큰 청크  : 맥락 풍부 / 노이즈·관련 없는 내용 섞임·임베딩 희석
해법: parent-document(작게 검색→크게 전달) 또는 auto-merging(인접 청크 자동 병합)
경험칙: 출발 256~512토큰 + overlap → 도메인서 [[Evaluation]]으로 측정해 조정
```

## 6. 결정 가이드

```
구조 있는 문서(md·HTML·코드) → document-structure 청킹부터
긴 서술형·참조 많음          → Contextual Retrieval 또는 late chunking으로 맥락 보강
검색 정밀 vs 맥락 충돌        → parent-document / auto-merging
무엇을 쓰든: 리랭킹을 얹으면 대부분 추가 이득(상위 후보 재정렬)
원칙: 청킹·맥락기법은 "측정 후 선택" — RAGAS·Recall@k로 도메인서 비교([[Evaluation]])
```

---

## 7. 관련
- [[RAG]] — 파이프라인·검색 품질 기법(§4)·실패 모드(§5)
- [[Embedding]] — 임베딩 모델·하이브리드(BM25+dense)·재색인
- [[Prompt-Caching]] — Contextual Retrieval 인덱싱 비용 절감(문서 캐시+청크 가변)
- [[GraphRAG]] — 청크 대신 엔티티/관계로 구조화하는 대안
- [[RAG-Latency-Reality]] — 구조화의 가치는 속도가 아닌 정확도
- [[Evaluation]] — 청킹 전략을 RAGAS·Recall@k로 비교 측정
- [[Reranking-Retrieval]] — 청크 검색 후 cross-encoder 재정렬(다음 단계)
- [[RAG-Ingestion-Pipeline]] — 청킹 앞단(파싱·정제·증분): 정제 품질이 청킹의 전제
- [[RAG-Failure-Debugging]] — 검색 누락(①) 진단 시 청크 크기·경계가 흔한 원인
- [[Code-RAG]] — 코드는 산문이 아닌 AST(함수/클래스) 경계로 청킹
