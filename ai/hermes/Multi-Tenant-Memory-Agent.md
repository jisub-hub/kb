---
tags:
  - ai
  - agent
  - architecture
  - memory
  - multi-tenant
  - rag
  - hermes
created: 2026-06-18
---

# 멀티테넌트 메모리 증강 에이전트

> [!summary] 한 줄 요약
> 사용자/팀 인증 → Hermes 호출 → 상호작용을 **개인·팀 지식베이스에 누적** → 다음 호출 때 검색·주입하여 점차 개인화되는 구조. 핵심은 ① "성장"은 **RAG 메모리(파인튜닝 아님)**, ② `tenant_id` 기반 **철저한 격리**(RLS), ③ **memory poisoning 방어**. 모델은 그대로, KB가 자란다.

---

## 1. ⚠️ "성장"의 두 의미 — 거의 항상 ①

```
① 메모리 증강 (RAG/메모리)   ← 99% 정답
   상호작용을 임베딩해 KB에 누적 → 다음 호출 때 검색·주입
   실시간·가벼움·즉시 반영. "persistent memory"가 이것.

② 파인튜닝 (가중치 학습)
   누적 데이터로 모델 재학습 → 무겁고 느림, 테넌트별 모델 분리 부담
   특수 케이스만.
```

> **모델이 똑똑해지는 게 아니라, 검색 가능한 기억이 풍부해진다.** Hermes는 두뇌로 고정, KB가 자란다. → 즉시 반영·테넌트 분리·망각(삭제)이 쉽다.

---

## 2. 아키텍처 흐름

```
[로그인] OAuth2/JWT → tenant_id(user_id/team_id) 추출
   │
   ▼
[질의 입력]
   │
   ▼
[KB 검색] tenant 필터로 본인/팀 기억 검색 (pgvector RAG)
   │  관련 기억 + 선호 + 과거 맥락
   ▼
[Hermes 호출] 검색 기억을 컨텍스트 주입 + function calling
   │  (OpenAI 호환 API)
   ▼
[응답]
   │
   ▼
[KB 저장] 상호작용을 임베딩해 tenant KB에 누적 ← "성장"
   │
   └─ 반복할수록 KB 풍부 → 개인화·정확도↑
```

---

## 3. 멀티테넌시 격리 (보안의 핵심)

> [!warning] 데이터 유출 = 치명적
> 한 사용자/팀의 기억이 다른 테넌트로 새면 안 된다. 격리는 **앱 로직이 아니라 DB 레벨로 강제**한다.

```
- 모든 KB 레코드에 tenant_id (user_id / team_id) 컬럼
- PostgreSQL Row-Level Security(RLS) 또는 쿼리 강제 필터
- 접근 키는 JWT의 tenant_id에서만 (앱이 임의 지정 금지)
- 개인 KB(user_id) + 팀 공유 KB(team_id) 계층 분리
```

```sql
-- tenant 격리 스키마 + RLS
CREATE TABLE memory (
  id          bigserial PRIMARY KEY,
  tenant_id   text NOT NULL,         -- user:123 또는 team:42
  scope       text NOT NULL,         -- 'personal' | 'team'
  kind        text NOT NULL,         -- 'fact' | 'preference' | 'episode'
  content     text NOT NULL,
  embedding   vector(1024),          -- bge-m3
  importance  real DEFAULT 0.5,
  created_at  timestamptz DEFAULT now()
);
CREATE INDEX ON memory USING hnsw (embedding vector_cosine_ops);

ALTER TABLE memory ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON memory
  USING (tenant_id = current_setting('app.tenant_id'));  -- 세션에 JWT tenant 주입
```

→ DB 격리 [[DB-Fundamentals]], 벡터 [[pgvector]] · [[Embedding]].

---

## 4. 메모리 계층

```
단기(세션):   현재 대화 히스토리 → 일정 길이 후 요약하여 장기로
장기(영속):
  - fact/preference: "이 사용자는 Spring을 쓴다" (구조화, 검색·주입)
  - episode:         과거 상호작용 임베딩 (RAG 검색)
  - team:            팀 공유 KB (문서·결정사항)
→ 호출 시 관련된 것만 선택 주입 (전부 X — 토큰·노이즈)
```

---

## 5. 구현 스케치 (Spring AI 흐름)

```java
@PostMapping("/chat")
public ChatResponse chat(@AuthenticationPrincipal Jwt jwt, @RequestBody ChatReq req) {
    String tenantId = jwt.getClaim("tenant_id");        // 1. 인증 → 테넌트
    setDbTenant(tenantId);                               //    RLS 세션 변수 주입

    // 2. tenant KB 검색 (RLS가 자동으로 본인 것만 반환)
    List<Memory> memories = memoryStore.searchSimilar(
        embed(req.message()), tenantId, topK(5));

    // 3. 기억을 컨텍스트로 주입 + Hermes 호출 (function calling)
    String answer = chatClient.prompt()
        .system("관련 기억:\n" + render(memories))        // 검색된 기억
        .user(req.message())
        .tools(toolRegistry)                              // function calling
        .call().content();

    // 4. 상호작용을 KB에 누적 (성장) — 선별 저장
    if (worthRemembering(req, answer)) {
        memoryStore.save(new Memory(tenantId, "episode",
            summarize(req, answer), embed(...)));         // 요약 후 저장
    }
    return new ChatResponse(answer);
}
```

> 인증 [[OAuth2-JWT]], 추론 [[Hermes-Agent]]·[[Local-LLM-API-Deployment]], 검색 [[RAG]]·[[RAG-Chatbot-E2E]].

---

## 6. 설계 주의점 (실무 사고 지점)

| 위험 | 설명 | 대응 |
|------|------|------|
| **테넌트 유출** | 기억이 다른 사용자에게 노출 | RLS·tenant 필터 강제(3절) |
| **Memory Poisoning** ⚠️ | 악의적 입력이 KB에 저장→다음 호출 오염(인젝션 지속화) | 저장 전 검증, 사용자발화/시스템사실 신뢰 경계 분리 |
| **메모리 노이즈** | 다 저장하면 검색 품질↓ | 중요도 점수·요약·중복 제거·선별 저장 |
| **기억 충돌** | 상반된 기억 누적 | 최신 우선·신뢰도 가중·주기적 정리 |
| **프라이버시** | 민감정보·삭제권(GDPR) | 보존기간·삭제 API·감사 로그 |
| **비용** | 매 호출 검색+주입→토큰↑ | 프롬프트 캐싱·요약 메모리 → [[LLMOps]] |

> Memory Poisoning은 자율 에이전트 보안의 핵심 — 저장된 기억이 미래 행동을 조종할 수 있다. [[Agent-Harness]] 가드레일, [[Security-Checklist]].

---

## 7. 스택 매핑 (vault 기준 — 이미 다 있음)

| 요소 | 구현 | 노트 |
|------|------|------|
| 인증/테넌트 | Spring Security + OAuth2/JWT | [[OAuth2-JWT]] |
| 추론 | Hermes (Ollama/vLLM, OpenAI 호환) | [[Hermes-Agent]] · [[Local-LLM-API-Deployment]] |
| KB 저장·검색 | pgvector + tenant_id + RLS | [[pgvector]] · [[Embedding]] |
| RAG | Spring AI | [[RAG]] · [[RAG-Chatbot-E2E]] |
| 에이전트 루프·도구 | function calling | [[Agent-Harness]] |
| 보안·격리 | RLS·승인 게이트·인젝션 방어 | [[Security-Checklist]] |
| 비용 | 캐싱·라우팅 | [[LLMOps]] |
| 자율 런타임(대안) | OpenClaw 등 | [[Autonomous-Agent-Runtimes]] |

---

## 8. 정리

```
가능하고 표준 패턴이다 (멀티테넌트 메모리 증강 에이전트).
핵심 3가지:
  ① "성장" = RAG 메모리 누적 (파인튜닝 아님) → 모델 고정, KB가 자람
  ② tenant_id 기반 격리 (RLS) → 데이터 유출 방지 (보안 1순위)
  ③ Memory Poisoning 방어 → 저장 데이터가 미래 호출을 오염

흐름: 인증(JWT tenant) → KB 검색(RLS) → Hermes(+기억 주입, 도구)
      → 응답 → 선별 저장 → 반복할수록 개인화↑
"학습"이 아니라 "기억" — 즉시 반영·테넌트 분리·망각이 쉽다.
```

---

## 관련
- [[Hermes-Agent]] — 추론·function calling
- [[Local-LLM-API-Deployment]] — Hermes를 OpenAI 호환 API로
- [[RAG]] · [[RAG-Chatbot-E2E]] · [[Embedding]] · [[pgvector]] — 지식베이스
- [[OAuth2-JWT]] — 인증·테넌트 식별
- [[Agent-Harness]] — 에이전트 루프·가드레일
- [[Autonomous-Agent-Runtimes]] — 런타임 대안(OpenClaw 등)
- [[Security-Checklist]] · [[LLMOps]] — 격리·비용 운영
