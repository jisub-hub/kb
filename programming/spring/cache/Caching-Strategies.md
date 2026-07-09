---
tags:
  - cache
  - strategy
  - performance
created: 2026-06-15
---

# 캐싱 전략 (Caching Strategies)

> [!summary] 한 줄 요약
> 캐시와 DB를 **어떻게 읽고 쓸지**에 대한 패턴. 가장 흔한 건 Look-aside(Cache-aside). 일관성·정합성 트레이드오프를 이해하고 골라야 한다.

---

## 1. 읽기/쓰기 전략

### Cache-Aside (Look-aside) — 가장 흔함
```
읽기: 캐시 확인 → (miss) DB 조회 → 캐시에 저장 → 반환
쓰기: DB 갱신 → 캐시 무효화(또는 갱신)
```
- ✅ 단순, 캐시 장애에도 DB로 동작. Spring `@Cacheable`이 이 방식.
- ❌ 첫 조회는 miss(콜드 스타트), 무효화 타이밍 관리 필요.

### Read-Through
- 캐시가 DB 조회를 **대신** 수행(애플리케이션은 캐시만 봄). 캐시 라이브러리/프록시가 처리.

### Write-Through
- 쓰기 시 **캐시와 DB를 동시에** 갱신 → 정합성↑, 쓰기 지연↑.

### Write-Behind (Write-Back)
- 캐시에 먼저 쓰고 DB는 **나중에 비동기** 반영 → 쓰기 빠름, 유실 위험.

---

## 2. 캐시의 3대 문제

| 문제 | 설명 | 대응 |
|------|------|------|
| **Cache Penetration(관통)** | 존재하지 않는 키 반복 조회 → 매번 DB | null도 짧게 캐싱, Bloom Filter |
| **Cache Avalanche(눈사태)** | 다수 키가 동시에 만료 → DB 폭주 | TTL에 지터(랜덤) 추가 |
| **Cache Stampede / Hot Key** | 인기 키 만료 순간 동시 요청 폭주 | 락(분산락)·재계산 단일화, 논블로킹 갱신 |

## 3. TTL & 무효화
- 모든 캐시에 **TTL 설정**(무한 캐시 지양).
- 쓰기 시 무효화(`@CacheEvict`) vs 갱신(`@CachePut`) 선택.
- [[CQRS]] read model을 캐시할 때는 이벤트로 무효화 트리거.

## 4. 베스트 프랙티스
- 캐시는 **정합성보다 성능** 트레이드오프 — "약간 stale 허용 가능한가?" 먼저 판단.
- 캐시를 **단일 진실 공급원으로 쓰지 말 것**(휘발 가정).
- 키 네이밍 규칙·버전 prefix로 대량 무효화 대비.

## 5. 관련
- [[Redis]] · [[Valkey]] · [[CQRS]]
