---
tags:
  - cache
  - moc
  - index
created: 2026-06-15
---

# ⚡ Cache MOC

> 분산 캐시 / 인메모리 데이터 스토어 정리.

- [[Redis]] — 사실상 표준 인메모리 데이터 스토어
- [[Valkey]] — Redis 포크(오픈소스, Linux Foundation). Redis 호환
- [[Caching-Strategies]] — 캐시 전략(Look-aside, Write-through 등) & 주의점

## 관련
- [[Spring-Cloud-Gateway]] (RateLimiter 저장소) · [[MSA]] · [[CQRS]] (read model 캐시)

---

## 🧭 한눈에
| | Redis | Valkey |
|---|-------|--------|
| 기원 | Redis Inc. | Redis 7.2.4 포크 (2024, LF/Valkey) |
| 라이선스 | RSALv2/SSPL(7.4~) | BSD (오픈소스) |
| 프로토콜 | RESP | RESP (호환) |
| 클라이언트 | Lettuce/Jedis | 동일 (호환) |
| 선택 이유 | 생태계·매니지드 풍부 | 완전 오픈소스 라이선스 회피 |
