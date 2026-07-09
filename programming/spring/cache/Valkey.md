---
tags:
  - cache
  - valkey
  - redis
  - in-memory
created: 2026-06-15
---

# Valkey

> [!summary] 한 줄 요약
> **Redis 7.2.4를 포크**한 오픈소스(BSD) 인메모리 스토어. 2024년 Redis가 라이선스를 RSALv2/SSPL로 바꾸자 Linux Foundation 주도로 출범. **Redis와 프로토콜·명령·클라이언트 호환**.

---

## 1. 배경
- 2024.3 Redis Inc.가 BSD → **RSALv2/SSPL** 로 라이선스 변경 (매니지드 사업자 견제 목적).
- 이에 반발해 커뮤니티/기업(AWS, Google, Oracle 등)이 마지막 BSD 버전(7.2.4)을 포크 → **Valkey**.
- Linux Foundation 산하, **BSD 라이선스 유지**.

## 2. Redis와의 관계
| | Redis | Valkey |
|---|-------|--------|
| 라이선스 | RSALv2/SSPL (7.4+) | **BSD (오픈소스)** |
| 프로토콜 | RESP | RESP (**호환**) |
| 명령어 | 동일 기반 | **호환** (7.2.4 기준 + 독자 기능 추가 중) |
| 클라이언트 | Lettuce/Jedis | **그대로 사용 가능** |
| 매니지드 | ElastiCache 등 | AWS ElastiCache/MemoryDB가 Valkey 지원 |

> [!tip]
> 대부분의 경우 **드롭인 대체** 가능 — 기존 Redis 클라이언트/설정을 거의 그대로 사용. Valkey는 멀티스레드 I/O 등 성능 개선도 진행 중.

## 3. Spring 사용법 (Redis와 동일)
```yaml
spring:
  data:
    redis:                      # 클라이언트 관점에선 redis 설정 그대로
      host: valkey-host
      port: 6379
```
```java
// Spring Data Redis(Lettuce/Jedis) 코드 변경 없이 Valkey 서버에 연결
@Cacheable(value = "product", key = "#id")
public Product getProduct(Long id) { ... }
```
> 자세한 사용 코드(@Cacheable, RedisTemplate, ZSet 랭킹, 분산락 등)는 [[Redis]] 노트 참고 — 그대로 적용된다.

## 4. 언제 Valkey를 고르나
- **라이선스 자유**가 중요한 경우(상용 배포·매니지드 제공).
- 완전한 오픈소스 거버넌스를 선호.
- 클라우드에서 Valkey 매니지드가 더 저렴할 때(예: ElastiCache for Valkey).

## 5. 관련
- [[Redis]] · [[Caching-Strategies]] · [[Spring-Cloud-Gateway]]
