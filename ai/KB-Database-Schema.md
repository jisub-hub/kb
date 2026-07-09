---
tags:
  - ai
  - rag
  - knowledge-management
  - database
  - postgresql
  - schema-design
  - normalization
  - permissions
created: 2026-06-19
---

# 사내 KB DB 스키마 설계 (정규화 + 권한 + 검색)

> [!summary] 한 줄 요약
> 마크다운 파일 KB를 prod DB로 옮길 때의 스키마. 원칙은 **진실원천은 정규화(FK·거버넌스), 검색경로는 비정규화로 materialize(tsv), 가변 메타는 jsonb**. 태그·taxonomy·권한·버전은 정규화하고, 검색은 단일 `chunks.tsv`로 평탄화한다. 관련: [[Obsidian-RAG-System]] · [[LLM-Auto-Tagging-Pipeline]] · [[RAG-Chatbot-E2E]] · [[Multi-Tenant-Memory-Agent]]

---

## 1. 핵심 원칙

> [!important] 다 정규화하면 핫패스가 6테이블 조인이 된다
> - **정규화**: 관계·거버넌스(태그/taxonomy/권한/버전) → FK 무결성·중복제거·통계·승인 워크플로
> - **비정규화(materialize)**: 검색용 `tsvector` → 인입 시 합성, 질의 땐 조인 없이 단일 테이블 스캔
> - **jsonb**: 가변·희소 속성(커넥터 config, 태깅 원본) → 컬럼 폭발 방지

검색은 어차피 [[RAG-Latency-Reality|병목이 아니지만]], 스키마가 핫패스를 다중 조인으로 만들면 권한 필터까지 겹쳐 느려진다. truth와 search representation을 분리한다.

## 2. 핵심 스키마 (Postgres)

```sql
-- 출처(커넥터)
sources(id, kind, name, config jsonb)              -- obsidian|confluence|drive...

-- 논리 문서 + 버전(내용 이력 → 재태깅/롤백)
documents(id, source_id FK, ext_id, path, title,
          current_version_id FK, created_at, deleted_at,
          UNIQUE(source_id, ext_id))
document_versions(id, document_id FK, content_hash,
                  body text, summary text,
                  aliases text[], hyde_questions text[],  -- 문서고유 자유텍스트(검색용)
                  created_at, UNIQUE(document_id, content_hash))

-- 검색 단위(청크) — 임베딩 도입 시 embedding 컬럼 추가
chunks(id, version_id FK, ord, content,
       tsv tsvector,                                -- ★ 검색 materialize
       UNIQUE(version_id, ord))
```

## 3. 태그 — 정규화의 핵심 (text[] 금지)

통제 어휘 + M:N 조인. 자유 text[]는 taxonomy 강제·중복제거·통계가 불가.

```sql
tags(id, slug UNIQUE, label, parent_id FK→tags,    -- 계층(taxonomy)
     status,                                        -- approved|proposed
     created_by, created_at)
tag_synonyms(tag_id FK, synonym, UNIQUE(synonym))   -- k8s→kubernetes 정규화

document_tags(document_id FK, tag_id FK, source,    -- source: human|llm
              confidence, added_by, created_at,
              PRIMARY KEY(document_id, tag_id, source))
```

→ FK로 어휘 강제, 사람/LLM 태그 구분·가중, 신규 태그 승인(`status='proposed'`). [[LLM-Auto-Tagging-Pipeline]]의 거버넌스가 여기에 얹힘.

## 4. 권한 — 그룹 우선 정규화 (FK 무결성 유지)

폴리모픽(`principal_type + principal_id`)은 FK가 깨진다. **그룹 기반 + 직접부여 예외**로 분리:

```sql
users(id, email UNIQUE, ...)
groups(id, name UNIQUE)
user_groups(user_id FK, group_id FK, PRIMARY KEY(user_id, group_id))

document_group_grants(document_id FK, group_id FK, perm,
                      PRIMARY KEY(document_id, group_id, perm))
document_user_grants (document_id FK, user_id  FK, perm,
                      PRIMARY KEY(document_id, user_id, perm))
-- 유효 접근 = 사용자의 그룹 ∪ 직접부여
```

ACL은 **원본 시스템 권한을 미러링**해 인입 시 채우고, 변경/삭제를 재동기화한다(권한 drift가 운영 난제).

## 5. 검색 materialize (비정규화가 정답인 곳)

`chunks.tsv`를 인입/태깅 시 필드 가중으로 합성 → 질의는 단일 테이블 + 권한 EXISTS만:

```sql
setweight(to_tsvector('korean', title), 'A')                       -- 최상
|| setweight(to_tsvector('korean', tags_text), 'B')
|| setweight(to_tsvector('korean', coalesce(summary,'') ||' '|| questions_text), 'C')
|| setweight(to_tsvector('korean', body), 'D')
```

권한 필터 결합 (pre-filter):

```sql
SELECT c.* FROM chunks c JOIN documents d ON d.id = ...
WHERE d.deleted_at IS NULL
  AND c.tsv @@ websearch_to_tsquery('korean', :q)
  AND ( EXISTS (SELECT 1 FROM document_group_grants g
                WHERE g.document_id=d.id AND g.group_id = ANY(:gids))
     OR EXISTS (SELECT 1 FROM document_user_grants u
                WHERE u.document_id=d.id AND u.user_id = :uid) )
ORDER BY ts_rank(...) DESC LIMIT :k;
```

한국어는 `korean`/`nori` analyzer로 색인·질의(조사·복합어).

## 6. 감사 / 운영

```sql
tagging_runs(id, version_id FK, model, prompt_version, raw_json jsonb, created_at)  -- 재현/롤백
sync_state(source_id FK, cursor, last_synced_at)                                    -- 증분 동기화
```

## 7. 방어심층 — RLS

앱에서 권한 pre-filter하되, **Postgres RLS**로 DB 레벨에서도 강제(다중 테넌트/ACL). 앱 버그가 있어도 DB가 막는다. → [[Multi-Tenant-Memory-Agent]]의 RLS 격리.

## 8. 인덱스

```sql
GIN(chunks.tsv);                                   -- FTS
btree(document_tags.tag_id);
btree(document_group_grants.group_id), (document_user_grants.user_id), (user_groups.user_id);
btree(documents.source_id, ext_id);
-- 벡터 도입 시: HNSW(chunks.embedding)
```

## 9. 과(過)정규화 경계 — 하지 말 것

- **별칭/Doc2Query 질문을 별 테이블로** → 개별 조회 안 하면 `text[]` + tsv로 충분 (필요해지면 child로 승격)
- **핫패스를 다중 조인으로** → tsv / materialized view로 평탄화
- **모든 메타를 컬럼화** → 가변·희소 속성은 `jsonb` 한 칸
- **권한 폴리모픽** → FK 무결성 위해 그룹/유저 테이블 분리

## 10. 요약 결정표

| 대상 | 전략 | 이유 |
|---|---|---|
| 태그·taxonomy | 정규화(테이블+FK) | 어휘 강제·중복제거·승인·통계 |
| 권한(user/group/ACL) | 정규화(조인) + RLS | 무결성·문서단위 격리 |
| 문서 버전 | 정규화(versions) | 재태깅·롤백·감사 |
| 검색(tsv) | 비정규화 materialize | 조인 없는 빠른 질의 |
| 별칭·질문·config | text[] / jsonb | 자유텍스트·가변속성 |

## 11. 관련 노트
- [[Obsidian-RAG-System]] — 이 DB를 쓰는 RAG 시스템 전체
- [[LLM-Auto-Tagging-Pipeline]] — `document_tags`/`tag_taxonomy`를 채우는 인입 파이프라인
- [[RAG-Chatbot-E2E]] — Spring AI + pgvector 구현 청사진
- [[Multi-Tenant-Memory-Agent]] — RLS 기반 테넌트/권한 격리
- [[RAG-Security]] — 권한 우회·데이터 유출 방어
- [[KB-Access-Control]] — §121 권한 pre-filter·RLS의 설계측: ACL 모델·검색 격리
- [[Structured-Data-RAG]] — 이 스키마를 자연어로 질의하는 검색측(text-to-SQL·스키마 링킹)
