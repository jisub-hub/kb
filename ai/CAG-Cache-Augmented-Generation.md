---
tags:
  - ai
  - rag
  - cag
  - prompt-caching
  - long-context
created: 2026-06-19
---

# CAG — Cache-Augmented Generation (검색 없는 지식 주입)

> [!summary] 한 줄 요약
> 지식 전체를 **컨텍스트에 통째로 넣고 그 prefix의 KV 캐시를 메모리에 고정**해, 검색(retrieval) 단계 없이 답하는 방식. "md 파일을 메모리에 올려둔다"의 **올바른 형태는 파일 텍스트가 아니라 처리 결과(KV)를 캐시하는 것**. 지식이 컨텍스트에 들어가고·자주 안 바뀔 때만 빛난다.

> 지도상 위치는 [[RAG-vs-Knowledge-Systems]] §8, KV 재사용 원리는 [[Prompt-Caching]], 이 볼트 적용은 [[Obsidian-RAG-System]]. 이 노트는 **CAG의 조건·함정·이 볼트 실측**.

---

## 1. RAG vs CAG — 무엇을 메모리에 두나

```
RAG: 매 질문마다 (검색된 청크 + 질문)을 LLM이 새로 prefill(attention 계산)
     → 검색 인프라 필요, 매번 prefill 재계산
CAG: 지식 전체를 컨텍스트에 1회 주입 → 그 prefix의 KV 캐시를 메모리에 고정
     → 검색 0, 이후 질문은 "질문 토큰"만 prefill, 지식 부분은 KV 재사용
```
> 핵심 오해 교정: **"파일을 RAM에 올려두기"는 효과 없다** — 디스크 읽기는 병목이 아니고(생성이 98%, [[RAG-Latency-Reality]]) OS 페이지 캐시가 이미 RAM에 둠. 의미 있는 캐시는 **KV 캐시**(=[[Prompt-Caching]]).

## 2. 성립 조건

```
① 지식이 컨텍스트에 들어갈 것 — Long-context 모델(수십만~1M 토큰)
② 지식이 자주 안 바뀔 것 — 바뀌면 그 뒤 prefix KV 캐시가 무효화(§4)
③ 질문 다수 / 같은 지식 반복 — 캐시 적재 비용을 질문 여러 번으로 분산(amortize)
→ "소규모 고정 지식 + 반복 질의"에 최적. 대규모·동적이면 RAG.
```

## 3. 이 볼트 실측 — CAG가 가능한가 ⭐

```
볼트 ≈ 2.0 MB / ~840K 토큰(보수 추정, 한·영 혼합이라 실제론 더 적을 것)
→ Opus 4.8 1M 컨텍스트에 "통째로" 들어감 → CAG 물리적으로 성립
→ 즉 "검색 없이 볼트 전체 + KV 캐시"가 이론상 가능
   ([[RAG-vs-Knowledge-Systems]] §7 Long-context가 RAG를 무너뜨리는 시나리오의 실현 조건)
```

## 4. ⚠️ 함정 — 공짜가 아니다

```
① 캐시 무효화: 파일 1개만 고쳐도 그 위치 "이후" prefix KV가 전부 무효
   → 이 볼트는 5분 루프로 끊임없이 수정됨 → 캐시가 계속 깨져 CAG 이점 증발 (치명적)
   완화: 자주 바뀌는 부분을 컨텍스트 "뒤쪽"에, 안정 부분을 "앞쪽"에 배치(앞 prefix 보존)
        — [[Agent-Persistent-Memory]] frozen snapshot이 prefix cache 지키는 원리와 동일
② Lost-in-the-middle: 840K를 통째로 넣으면 중간 정보를 모델이 놓침 → 정확도↓
   → "전부 넣기"가 "선택해서 넣기"(LLM Wiki/RAG)보다 정확도엔 불리할 수 있음
③ KV 캐시 메모리: 로컬 모델은 KV가 VRAM을 크게 먹음
   (70B·840K 토큰 KV ≈ 수십 GB) → 단일 GPU엔 부담 ([[LLM-System-Performance-Analysis]])
④ 캐시 TTL·저장 비용: API 프롬프트 캐시는 TTL(예 5분)·쓰기 비용 있음([[Prompt-Caching]])
```

## 5. 결정 — 이 볼트엔 무엇이 맞나

```
자주 수정 + 정확도 중요(현재 상태):
  → CAG ❌ (무효화로 이점 증발 + lost-in-middle)
  → LLM Wiki식 선택(인덱스→관련 노트만) + 그 작은 컨텍스트에 프롬프트 캐시 ⭐
     ([[Obsidian-RAG-System]]·[[LLM-Wiki]])
고정 스냅샷 + 대량 질의(볼트를 며칠 안 고치고 집중 질문):
  → CAG가 빛남 — 검색 0·prefill 재계산 0, 안정 prefix면 무효화도 없음
파일 RAM 캐싱:
  → 구현 불필요(병목 아님·OS가 함)
```

## 6. CAG vs RAG vs LLM Wiki 한 줄

```
RAG      : 큰·동적 지식 → 청크 검색해 그때그때 주입
LLM Wiki : 중간·구조화 지식 → 인덱스로 선택해 파일 주입([[LLM-Wiki]])
CAG      : 작은·고정 지식 → 전부 주입 + KV 캐시(검색 없음)
→ 세 방식 모두 "비파라메트릭"(가중치 안 건드림). 파라메트릭은 [[Parametric-Knowledge-Editing]].
```

---

## 7. 관련
- [[RAG-vs-Knowledge-Systems]] — CAG의 지도상 위치(§8)·Long-context 시나리오(§7)
- [[Prompt-Caching]] — CAG의 핵심 메커니즘(입력 KV 재사용·TTL·비용)
- [[Obsidian-RAG-System]] — 이 볼트의 실제 선택(CAG 아닌 선택+캐시)
- [[LLM-Wiki]] — 선택적 주입(CAG의 "전부 주입" 대안)
- [[RAG-Latency-Reality]] — 병목은 생성이지 디스크/검색이 아님
- [[Agent-Persistent-Memory]] — frozen snapshot이 prefix cache 보존하는 동일 원리
- [[LLM-System-Performance-Analysis]] — KV 캐시 메모리(VRAM) 비용
