---
tags:
  - ai
  - llm
  - knowledge-base
  - rag
  - obsidian
created: 2026-06-19
---

# LLM Wiki — 검색 레이어 없이 만드는 지식 베이스 (구축 가이드)

> [!summary] 한 줄 요약
> 벡터DB·임베딩 없이 **마크다운 파일 + 인덱스 + 링크**로 만드는 KB. LLM이 인덱스를 읽고 필요한 파일을 **선택해 통째로** 컨텍스트에 넣는다 — RAG의 임베딩이 하던 "검색"을 **구조(인덱스·설명·링크)가 대신**한다. 핵심 명제: **LLM Wiki를 만든다 = LLM이 읽을 것을 전제로 구조를 설계한다.** (이 Obsidian vault가 그 사례)

> "LLM Wiki가 RAG인가"의 이론·위치는 [[RAG-vs-Knowledge-Systems]], 검색 단위 비교는 그 §11.5. 이 노트는 **"어떻게 구축·운영하나"**의 실천 가이드.

---

## 1. 작동 원리 — 구조가 검색 엔진이다

```
RAG:      질문 → 임베딩 → 벡터 유사도 top-k 청크 → 컨텍스트   (확률적·조각)
LLM Wiki: 질문 → LLM이 [인덱스] 읽기 → 파일 선택 → 그 파일(+링크) 읽기 → 컨텍스트
                 └ 결정적·설명가능(왜 그 파일인지)  └ wikilink 순회 = 수동 GraphRAG
```
> 같은 패턴이 [[Hermes-Skills]] skills_list→skill_view, [[Tool-Search]] 3브리지 툴, [[Agent-Persistent-Memory]] MEMORY.md에 반복된다 — **progressive disclosure**(필요할 때만 펼침).

## 2. 4가지 구성요소 (= 만드는 법)

```
① 인덱스(MOC)   — 검색 진입점. LLM이 맨 먼저 읽음
   `_index.md`에  - [[노트]] — 한 줄 설명  형식
   ⭐ 이 "설명"이 곧 선택 신호(=RAG의 임베딩 역할). 모호하면 LLM이 엉뚱한 파일 선택
② 파일          — 검색 단위. "한 개념 = 한 파일", 제목 명확
   파일 통째가 컨텍스트로 → 자기완결(앞에 한 줄 요약 callout), 적당 크기
③ wikilink      — 그래프 엣지. 관련 노트를 [[양방향]] 연결
   → LLM이 한 파일 읽고 링크 따라 확장 = 관계 순회를 공짜로([[GraphRAG]]의 수동 버전)
④ 계층(폴더·태그) — 범위 좁히기. 도메인별 _index → 최상위 인덱스(README)
```

## 3. LLM이 실제로 접근하는 경로

```
A. 파일 도구 있는 에이전트(Claude Code·Hermes): read_file로 인덱스→파일 자율 탐색
B. 도구 없음(2-pass): 인덱스 먼저 제시 → "어느 파일?" → 해당 파일 붙여넣기
C. git 보관: 버전·diff·협업 + 에이전트가 git으로 읽기
이점: 큰 컨텍스트(128K~1M)에 "선택된 몇 파일 통째" → 청크 조각보다 맥락 보존 우수
```

## 4. 잘 만드는 규칙 (RAG와 다른 점)

| 규칙 | 이유 |
|------|------|
| **인덱스 설명을 정성껏** | 이게 검색 품질 결정(임베딩 대신). 가장 중요 |
| **한 개념 한 파일 + 자기완결 요약** | 파일이 통째 들어가므로 |
| **양방향 링크** | 단방향이면 한쪽에서 못 찾음 |
| **바운디드 큐레이션** | 무한정 안 쌓고 정리(→ [[Agent-Persistent-Memory]] 철학) |
| **일관된 하우스스타일** | 제목·섹션·태그 규칙 → LLM이 패턴으로 빠르게 파싱 |
| **⚠️ 규모 ~수백 페이지** | 넘으면 인덱스 비대 → 벡터 RAG 폴백 얹기(§6) |

## 5. 언제 LLM Wiki / 언제 넘어가나

```
LLM Wiki로 충분: 개인·팀 KB, 수백 노트 이하, 자주 편집, 디버깅 중요 (← 이 vault)
벡터 RAG 추가  : 노트가 수천+ 이거나 "제목 모르는 퍼지 의미검색"이 잦을 때 → 하이브리드
GraphRAG 자동화: wikilink 수동 그래프로 충분 → 자동 엔티티 추출은 ROI 안 나옴(대부분)
파라메트릭     : 별개 축(가중치) → [[Parametric-Knowledge-Editing]]
전체 비교·선택트리: [[RAG-vs-Knowledge-Systems]] §11.5
```

## 6. 확장 — Wiki 우선 하이브리드

```
1차: LLM Wiki(구조로 선택, 파일 통째)
보강: 규모↑/의미검색 필요 → bge-m3 임베딩 + pgvector 벡터 폴백([[Embedding]])
관계: wikilink 그래프 탐색(자동 GraphRAG 불필요)
능동: LLM이 추가 탐색 결정(Agentic) → [[Agent-Harness]]
→ "잘 만든 LLM Wiki = naive RAG보다 나은 RAG"([[RAG-vs-Knowledge-Systems]] §11)
```

## 7. 이 vault에 대입 (실사례)

```
이미 갖춤: _index MOC · wikilink 그래프 · 태그 · README 최상위 인덱스 · git
운영으로 강화: 양방향 역링크 촘촘화 · 하우스스타일 통일 · 깨진 링크 0 유지
            (반자율 강화 루프 REVIEW-QUEUE가 매일 하는 일 = LLM Wiki 구축 그 자체)
다음 단계: ① 각 _index 설명을 더 검색친화적으로  ② 수천 노트 되면 벡터 폴백
```
> 핵심: LLM Wiki "구축"은 벡터DB 세우기가 아니라 **인덱스·파일·링크를 LLM 친화적으로 설계**하는 일. 매일의 구조화(역링크·_index·스타일)가 곧 검색 품질 투자다.

---

## 8. 관련
- [[RAG-vs-Knowledge-Systems]] — LLM Wiki의 정의·위치(§6)·"RAG인가"·4축 비교(§11.5)
- [[Agent-Persistent-Memory]] — 바운디드 큐레이션·frozen snapshot(메모리판 LLM Wiki)
- [[Hermes-Skills]] — description 선택신호·progressive disclosure(같은 패턴)
- [[GraphRAG]] — wikilink가 곧 수동 knowledge graph
- [[Embedding]] — 규모 확장 시 벡터 폴백
- [[Agent-Harness]] — 에이전트가 Wiki를 능동 탐색(Agentic)
- [[Obsidian-RAG-System]] — 이 볼트를 one-shot RAG로 구현한 사례(LLM Wiki + 벡터 폴백)
