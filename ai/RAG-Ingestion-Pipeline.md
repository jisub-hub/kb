---
tags:
  - ai
  - rag
  - ingestion
  - etl
  - retrieval
created: 2026-06-22
---

# RAG 인입 파이프라인 운영 (Ingestion / ETL)

> [!summary] 한 줄 요약
> RAG 품질의 **상류 천장**. 검색·리랭킹·생성을 아무리 다듬어도 **인입 단계에서 망가진 텍스트·오래된 청크·누락 문서**는 복구 불가("garbage in → garbage retrieval"). 청킹([[Chunking-Strategies]])이 *어떻게 자를지*라면, 이 노트는 그 앞단 — *무엇을 어떻게 들여와·갱신·삭제할지*의 운영 난제.

> 분할은 [[Chunking-Strategies]], 모델 교체 재색인은 [[Embedding]] §7, happy-path 코드는 [[RAG-Chatbot-E2E]] §3. 이 노트는 **인입의 운영 hard parts**(파싱·증분·삭제·메타).

---

## 1. 왜 — 인입이 검색 천장을 정한다

```
인입 파이프라인:
  소스(PDF/HTML/DB/위키) → 로드 → 파싱·정제 → 청킹 → 임베딩 → 벡터DB 적재
  └ 여기서 망가지면 ↓ 모든 하류가 그 위에서 동작(검색·생성이 못 살림)
실패 예: PDF 표가 줄바꿈으로 뭉개짐 → 청크가 의미 단위 아님 → 임베딩 엉뚱 → 검색 불능
원칙: retrieval 정확도의 상한은 인입 품질이 정한다. 검색 튜닝 전에 인입부터 본다.
```

## 2. 파싱 품질 — garbage in 차단

```
문서 타입별 함정:
  PDF      : 다단(컬럼) 순서 꼬임·표 구조 소실·머리말/꼬리말 반복 삽입·스캔본=이미지
  HTML     : 네비/광고/푸터 보일러플레이트가 본문에 섞임 → 노이즈 청크
  Office   : 표·각주·변경이력 추출 누락
정제: 보일러플레이트 제거·표는 구조 보존(markdown table/HTML)·헤더풋터 필터
스캔/이미지 PDF: OCR 필요 → 품질 낮으면 [[Multimodal-RAG]] OCR-free 시각 검색으로 우회
검증: 파싱 직후 샘플 육안 점검·빈 청크/깨진 인코딩 비율 모니터(인입 QA 게이트)
```

## 3. 증분 동기화 (incremental sync)

```
전체 재색인(나이브): 매번 전부 다시 → 비용·시간↑, 대규모서 비현실
증분: 변경분만 처리
  변경 감지 : content hash(본문 해시)·etag·last-modified·source 버전
  흐름      : 소스 스캔 → hash 비교 → (신규/변경)만 재청킹·재임베딩 → upsert
  중복 제거 : 동일 본문 다중 경로 → hash로 dedup(같은 청크 중복 적재 방지)
키 설계: 문서ID + 청크 인덱스(또는 청크 hash)를 안정적 PK로 → upsert/삭제 추적 가능
```

## 4. ★삭제·갱신 전파★ — 가장 흔한 운영 사고 ⚠️

```
실패 모드: 원본이 삭제·수정됐는데 벡터DB에 옛 청크가 남음(stale)
  → 검색에 잡혀 "이미 폐기된 정보"로 답함(조용한 환각·규정 위반)
전파 규칙:
  소스 삭제 → 해당 문서의 모든 청크 삭제(tombstone/하드 삭제)
  소스 수정 → 옛 청크 삭제 후 새 청크 적재(부분 upsert면 잔여 청크 누락 주의:
             문서가 5청크→3청크로 줄면 옛 4·5번 청크를 반드시 제거)
권한 변경 → 즉시 반영(접근 불가가 된 문서가 검색되면 유출 — [[KB-Access-Control]])
구현: 문서ID로 청크를 묶어 "문서 단위 삭제 후 재적재"가 가장 안전(부분 갱신보다 사고↓)
```

## 5. 메타데이터 추출 — 하류의 입력

```
인입 시 청크에 메타 부착 → 하류 기능의 재료:
  출처(문서명·URL·페이지) → [[Citation-Attribution]] 인용 단위
  시점(작성·갱신일)        → 신선도 필터·시간 가중
  권한(소유자·그룹·분류)   → 검색 pre-filter·[[RAG-Security]]·[[KB-Access-Control]] RLS
  타입/언어/태그           → 라우팅([[Query-Transformation]])·하이브리드 필터
원칙: 검색·인용·권한에 쓸 메타는 "인입 시점에" 확정(나중에 소급 부여는 비쌈).
자동 강화: 인입 시 LLM이 태그·요약·질문 추출([[LLM-Auto-Tagging-Pipeline]])
```

## 6. 트레이드오프

```
실시간 동기화(CDC/웹훅) vs 배치(주기 크롤):
  실시간 : 신선도↑·복잡도/비용↑(이벤트 유실·순서 처리)
  배치   : 단순·저비용·신선도 지연(야간 재색인 등)
인입 지연 vs 신선도: 무거운 파싱·LLM 강화는 인입 throughput↓ → 우선순위/큐로 분리
정합성: 소스↔벡터DB 주기적 reconciliation(누락·고아 청크 감사)로 drift 점검
원칙: 대부분 배치+증분으로 충분. 실시간은 "최신성이 SLA인" 도메인에만.
```

## 7. RAG 데이터 흐름에서의 위치

```
인입(이 노트) : 로드·파싱·정제·증분·삭제·메타  ← 품질 상류 천장
  → 청킹      : 분할 전략([[Chunking-Strategies]])
  → 임베딩    : 벡터화·재색인([[Embedding]])
  → 검색      : 리랭킹·쿼리변환·필터
  → 생성·귀속 : 답·인용([[RAG]]·[[Citation-Attribution]])
→ 하류 전부가 인입 위에서 동작. 인입이 정확도의 ceiling.
```

---

## 8. 관련
- [[Chunking-Strategies]] — 인입 다음 단계(분할). 인입 정제가 청킹 품질의 전제
- [[Embedding]] — 적재·모델 교체 재색인(§7), 인입의 임베딩 단계
- [[RAG-Chatbot-E2E]] — DocumentLoader·ingest happy-path 코드 스케치(§3)
- [[Citation-Attribution]] — 인입 메타(출처)가 인용 단위의 재료
- [[RAG-Security]] · [[KB-Access-Control]] — 권한 메타·삭제 전파(접근불가 문서 유출 방지)
- [[Multimodal-RAG]] — 스캔/이미지 PDF는 OCR-free 시각 검색으로 우회
- [[LLM-Auto-Tagging-Pipeline]] — 인입 시점 LLM 메타 강화
- [[RAG]] — 전체 파이프라인(인입은 그 상류)
- [[RAG-Failure-Debugging]] — 파싱 유실·정제 누락이 검색 recall(①) 실패의 상류 원인
