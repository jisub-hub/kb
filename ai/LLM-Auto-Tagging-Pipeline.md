---
tags:
  - ai
  - rag
  - llm
  - tagging
  - metadata
  - knowledge-management
  - ingestion
  - taxonomy
created: 2026-06-19
---

# LLM 자동 태깅 파이프라인 (색인 시점 의미 강화)

> [!summary] 한 줄 요약
> 문서 인입 시 LLM으로 **태그·별칭·요약·"이 문서가 답하는 질문"(Doc2Query)** 을 추출해 색인에 박아두는 파이프라인. 의미 작업을 질의 시점(임베딩)이 아니라 **색인 시점**으로 옮겨, 키워드/BM25 검색만으로 의미 검색에 근접한다. 핵심은 **통제 어휘(taxonomy) + 사람 승인 루프**로 태그 폭발을 막는 것. 관련: [[Obsidian-RAG-System]] · [[Chunking-Strategies]] · [[LLM-Wiki]] · [[RAG-Security]]

---

## 1. 왜 색인 시점 태깅인가

벡터 임베딩은 질의 시점에 의미를 매칭한다(인프라·재인덱싱·불투명성 비용). 대신 **인입 시 LLM이 문서의 의미를 사람이 읽을 수 있는 필드(태그/요약/질문)로 추출**해 두면, 질의 시점엔 가벼운 BM25만으로도 의미를 잡는다.

- 설명가능·거버넌스 용이(불투명 벡터와 대비) → 사내 KB 감사/정합성에 유리
- 권한 pre-filter와 궁합 좋음, GPU 상시 불필요(인입 시 1회)
- 한계: 완전히 새로운 패러프레이즈는 dense 벡터만 못함, **taxonomy 유지가 사람 일**

## 2. 파이프라인 단계

```
문서 인입/수정
  → (1) 변경 감지(content hash)         # 이미 같은 버전이면 skip(멱등)
  → (2) 전처리(텍스트 추출·청크·길이 컷)
  → (3) LLM 추출(구조화 JSON)            # 태그·별칭·엔티티·요약·질문·키워드
  → (4) Taxonomy 매핑                    # 기존 어휘로 정규화, 신규는 'proposed'
  → (5) 정규화·중복 병합                  # 소문자/kebab, 동의어 머지
  → (6) DB 반영 + FTS tsv 재생성(필드 가중)
  → (7) 사람 검토 큐(신규 태그·taxonomy 승인)
```

재태깅 트리거: 본문 해시 변경 시 (3)부터 재실행. 버전 보존.

## 3. LLM 추출 (구조화 출력)

문서당 1회, **JSON 스키마 강제 + few-shot + 현재 taxonomy 주입**으로 일관성 확보.

```jsonc
{
  "tags": ["kafka", "exactly-once", "outbox"],   // taxonomy에서 우선 선택
  "new_tags": ["transactional-outbox"],          // 없으면 제안(승인 전까지 proposed)
  "aliases": ["EOS", "정확히 한 번", "메시지 중복 제거"], // 동의어·약어·영한
  "entities": ["Kafka", "Debezium"],
  "summary": "Kafka에서 Exactly-Once를 보장하는 패턴...", // 1~3문장
  "hyde_questions": [                              // ★ 이 문서가 답하는 질문 3~5개
    "Kafka에서 메시지 중복을 어떻게 막나?",
    "Outbox 패턴은 언제 쓰나?"
  ],
  "keywords": ["멱등 프로듀서", "트랜잭션 코디네이터"]
}
```

> [!tip] Doc2Query가 핵심
> "이 문서가 답하는 질문"을 생성해 색인하면, 사용자 질의와 **직접 매칭**된다. 순수 키워드 시스템에서 의미 recall을 얻는 정석(Doc2Query/docTTTTTquery). 질의 시점 LLM 확장과 합쳐 양방향 강화.

추출 프롬프트 원칙: "제공된 taxonomy 목록에서 우선 고르고, 정말 없을 때만 new_tags 제안", "본문 근거 없는 태그 금지", "출력은 JSON만".

## 4. Taxonomy 거버넌스 (안 하면 망가짐)

> [!warning] 자유 생성 태그는 폭발·드리프트한다
> 통제 없이 두면 동의어·표기변형이 난립한다(개인 볼트도 572 태그까지 증가). 반드시 통제 어휘를 둔다.

- **통제 어휘 테이블**: 승인된 태그만 1급. LLM은 매핑 우선, 신규는 `proposed`.
- **신규 태그 승인 큐**: 사람이 승인/반려/병합. 승인 시 `approved` 편입.
- **중복 검사**: 신규 태그를 기존과 문자열 유사도(+선택적으로 임베딩)로 비교해 자동 병합 후보 제시.
- **정규화 규칙**: 소문자, kebab-case, 동의어 사전(`k8s`→`kubernetes`).
- **출처·신뢰도**: `source ∈ {human, llm}`, `confidence`. 사람 태그를 검색 가중·신뢰에서 우대.

## 5. 데이터 모델 (Postgres, [[Obsidian-RAG-System]] 스키마 확장)

```sql
documents(id, source, ext_id, title, path, content_hash,
          summary text, aliases text[], hyde_questions text[],
          created_at, updated_at, deleted_at)

document_tags(document_id, tag, source, confidence)   -- source: human|llm
tag_taxonomy(tag, parent, status, synonyms text[])    -- status: approved|proposed
tagging_runs(document_id, content_hash, model, created_at, raw_json)  -- 감사/재현

-- 검색용 가중 tsv (인입 시 생성)
-- setweight(title,'A') || setweight(tags+aliases,'B')
--        || setweight(summary+hyde_questions,'C') || setweight(body,'D')
```

## 6. 검색 통합 (임베딩 없이)

BM25 필드 가중으로 색인된 의미 필드를 활용:

```
title^3, tags^2.5, aliases^2, hyde_questions^2, summary^1.5, body^1
```

- 한국어는 **`nori`(형태소 분석기)** 로 색인/질의 → 조사·복합어 처리
- 질의 시점 **LLM 질의 확장**(동의어·영한 대체어 생성 후 합집합)과 결합 → 양방향 의미 강화
- 권한은 [[Obsidian-RAG-System]]의 ACL pre-filter 그대로

## 7. 운영

- **멱등성**: `content_hash` 동일하면 재태깅 skip. 변경분만 처리(증분).
- **배치/비용**: 인입 시 문서당 LLM 1회(버전당). 4090 + 로컬 모델이면 대량도 저렴. 실패 시 재시도 큐, 부분 실패 격리.
- **재현/감사**: `tagging_runs`에 모델·해시·원본 JSON 보존 → 품질 디버깅·롤백.
- **모델**: 구조화 출력 잘하는 instruct 모델(예: 로컬 gemma4-26b / qwen3). thinking은 off 권장(속도·결정성).

## 8. 평가 & 개선

- 샘플 문서에 사람 정답 태그 vs LLM 태그 → **precision/recall** 측정. → [[Evaluation]]
- 질의 로그로 "어떤 태그/질문이 실제 히트에 기여했나" 분석 → 프롬프트·가중치 튜닝.
- 태그 사용 빈도 모니터링: 거의 안 쓰이는 태그 정리, 과다 태그 분할.

## 9. 트레이드오프 요약

| | LLM 자동 태깅(색인 강화) | 벡터 임베딩 |
|---|---|---|
| 의미 매칭 | 태그·요약·Doc2Query로 상당 부분 | 강함(연속 공간) |
| 설명가능성 | 높음(사람이 읽는 태그) | 낮음(불투명) |
| 인프라 | BM25만, 벡터DB 불필요 | 임베딩모델+벡터DB+재인덱싱 |
| 유지비용 | taxonomy 큐레이션(사람) | 인프라(머신) |
| 권한 필터 | 깔끔 | filtered-ANN 주의 |

→ 키워드+태그 KB의 **의미 보강 1순위**. 부족분이 평가로 드러나면 그때 하이브리드로 벡터를 얹는다. → [[Obsidian-RAG-System]]

## 10. 관련 노트
- [[Obsidian-RAG-System]] — 이 태깅이 들어갈 RAG 시스템 전체(검색·권한·배포)
- [[Chunking-Strategies]] — 청킹(긴 문서 추출 전처리)
- [[LLM-Wiki]] — 구조=검색엔진, 검색 레이어 없는 KB
- [[Evaluation]] — 태깅/검색 품질 평가(RAGAS)
- [[RAG-Security]] — 비신뢰 콘텐츠·프롬프트 인젝션(인입 LLM도 대상)
- [[Prompt-Engineering]] — 구조화 출력·few-shot 프롬프트 설계
