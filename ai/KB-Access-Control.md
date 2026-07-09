---
tags:
  - ai
  - rag
  - knowledge-management
  - security
  - acl
  - permissions
  - rbac
  - multi-tenant
created: 2026-06-22
---

# 사내 KB 권한 설계 (ACL / 접근 제어)

> [!summary] 한 줄 요약
> ACL = "누가(주체) 어떤 문서(자원)에 어떤 권한"인지의 목록. KB RAG의 핵심 철칙은 **검색 단계에서 pre-filter** — 사용자가 볼 수 있는 문서만 후보로 삼아야 권한 없는 지식이 답변·인용·캐시로 새지 않는다. PG=권한 진실원천, ES로 검색하면 ACL을 ES에 미러링해 거기서 필터. 관련: [[KB-Database-Schema]] · [[RAG-Security]] · [[Multi-Tenant-Memory-Agent]]

---

## 1. ACL이란
**Access Control List** — 3요소로 구성:
- **주체(Principal)**: 사용자(user) / 그룹(group)
- **자원(Resource)**: 문서(KB에선 document)
- **권한(Permission)**: read / write / delete (검색은 보통 `read`)

```
문서 "급여규정"의 ACL:
  group:인사팀   → read
  group:경영지원 → read
  user:hong      → read   (개별 예외)
```

## 2. 모델 선택
- **그룹 기반(RBAC 결합)**: 사용자마다 부여하지 말고 **그룹에 부여** → 운영 단순. 사용자는 그룹 멤버십으로 권한 획득.
- **allow-list 우선**: 기본은 차단, 허용만 명시. deny 혼용은 평가 복잡 → 가급적 지양.
- **상속**: 폴더/Confluence space/Drive 공유 권한을 문서가 상속 → 원본 시스템 ACL을 **미러링**.

## 3. 데이터 모델 (→ [[KB-Database-Schema]])
```sql
users(id, email, ...)
groups(id, name)
user_groups(user_id, group_id)                       -- 멤버십(SSO 동기화)

document_group_grants(document_id, group_id, perm)    -- 그룹 부여(주력)
document_user_grants (document_id, user_id,  perm)    -- 개별 예외
-- 유효 접근 = 사용자의 그룹 ∪ 직접부여
```

## 4. RAG에서의 철칙 — pre-filter

> [!important] 검색 후 제거(post-filter)가 아니라 검색 시 pre-filter
> 권한 필터를 **검색 쿼리에 밀어넣어**, 애초에 허용 문서만 후보가 되게 한다. post-filter는 (a) recall 손상 (b) 캐시/로그/인용으로 누출 위험.

사용자 principal = `{user_id} ∪ {group_ids}` 를 검색에 결합:

```sql
-- PG(pgvector/FTS) 단일 저장소일 때
... WHERE c.tsv @@ :q
  AND ( EXISTS(SELECT 1 FROM document_group_grants g
              WHERE g.document_id=d.id AND g.group_id = ANY(:gids))
     OR EXISTS(SELECT 1 FROM document_user_grants u
              WHERE u.document_id=d.id AND u.user_id = :uid) )
```

## 5. PG + ES 분리 시 — ACL 미러링
문서/청크가 ES에 있고 권한 진실원천이 PG면:
- 각 ES 문서에 `allowed_groups[]`, `allowed_users[]` 필드를 **미러링**
- 검색 시 사용자 principal로 **ES 단에서 필터**(document-level security)
- **PG=진실원천 → 권한 변경 시 ES로 증분 동기화**
- 흐름: `PG(사용자→그룹) → ES(검색+ACL필터 top-k) → 앱 조립 → LLM`

## 6. 방어심층 — RLS
앱에서 pre-filter하되 **Postgres RLS**로 DB 레벨에서도 강제 → 앱 버그가 있어도 DB가 막음(다중 테넌트). → [[Multi-Tenant-Memory-Agent]]

## 7. ACL 출처 & 동기화 (운영 난제)
- 원본 시스템 권한(Confluence/Drive/사내 그룹)을 **인입 시 미러링**
- **권한 drift**: 원본에서 권한 바뀌면 KB도 재동기화(webhook/주기). 안 하면 옛 권한으로 노출
- **삭제 전파**: 문서·권한 회수 시 인덱스에서 즉시 제거

## 8. 함정 체크리스트
> [!warning]
> - **post-filter 금지** — 반드시 pre-filter
> - **캐시는 권한집합 단위로 키잉** — 전역 응답캐시 = 다른 사용자에게 누출
> - **인용/스니펫도 권한 필터 후** 생성 — 안 보이는 문서가 출처로 새면 안 됨
> - **filtered-ANN 주의** — 벡터검색에 제한적 ACL pre-filter는 recall/속도 저하 → filtered HNSW / 파티셔닝
> - **MCP/에이전트 신원 전파** — 공유 서비스계정이 아니라 **호출자 신원**으로 ACL 적용
> - **권한 drift / 삭제 전파** 동기화 누락 = 가장 흔한 유출 경로

## 9. 관련 노트
- [[KB-Database-Schema]] — ACL 테이블이 포함된 전체 스키마
- [[RAG-Security]] — 데이터 유출·프롬프트 인젝션 방어 전반
- [[Multi-Tenant-Memory-Agent]] — RLS 기반 테넌트/권한 격리
- [[Obsidian-RAG-System]] — 이 권한이 적용될 RAG 시스템
