---
tags:
  - concurrency
  - distributed-lock
  - redis
  - redisson
  - database
created: 2026-06-17
---

# 분산 락 (Distributed Locking)

> [!summary] 한 줄 요약
> 여러 서버/프로세스가 공유 자원에 동시 접근할 때, **JVM을 넘어선 상호 배제**를 보장하는 락. `synchronized`는 단일 인스턴스 안에서만 유효하므로 분산 환경에선 DB·Redis 같은 **공유 저장소**에 락을 둔다. 단, "가능하면 락을 쓰지 않는" 설계가 우선.

---

## 1. 왜 단일 JVM 락이 분산에서 깨지는가
```java
// ❌ 인스턴스가 2개 이상이면 무의미
public synchronized void decreaseStock(Long id) { ... }
```
- `synchronized`·`ReentrantLock`은 **그 JVM 힙 안의 모니터**만 잠근다. 같은 코드가 서버 A·B에서 동시에 돌면 둘 다 락을 "획득"했다고 믿고 재고를 동시에 깐다 → **갱신 손실(race condition)**.
- 오토스케일링([[../deploy/Kubernetes]])으로 파드가 늘어나는 환경에선 거의 항상 문제. 락의 범위를 **모든 인스턴스가 공유하는 곳**으로 끌어올려야 한다.

## 2. 동시성 제어의 종류 — 락이 정말 필요한가부터
DB 한 행에 대한 경합이라면 분산 락보다 **DB 락**이 더 단순하고 안전할 때가 많다.

| 방식 | 동작 | 충돌 빈도 | 장점 | 단점 |
|------|------|-----------|------|------|
| **낙관적 락** `@Version` | 버전 불일치 시 예외, 재시도 | 낮을 때 ⭐ | 락 대기 없음, 처리량↑ | 충돌 잦으면 재시도 폭증 |
| **비관적 락** `SELECT ... FOR UPDATE` | 행을 미리 잠금 | 높을 때 ⭐ | 확실한 직렬화 | 락 대기·데드락 위험, 커넥션 점유 |

```java
// 낙관적 락 — 엔티티에 @Version
@Entity
class Stock {
    @Id Long id;
    int quantity;
    @Version Long version;   // UPDATE ... WHERE id=? AND version=? → 0건이면 OptimisticLockException
}

// 비관적 락 — JPA
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select s from Stock s where s.id = :id")
Stock findByIdForUpdate(@Param("id") Long id);   // SELECT ... FOR UPDATE
```
- 충돌이 드물면 낙관적 락 + 재시도가 최선. 짧고 강한 경합이면 비관적 락. 자세한 동작은 [[JPA-Hibernate]].

## 3. DB 기반 분산 락
DB 자체를 락 매니저로 사용 — 별도 인프라가 필요 없다.

```sql
-- MySQL named lock (커넥션 단위, 세션이 끊기면 자동 해제)
SELECT GET_LOCK('stock:42', 3);   -- 3초 타임아웃, 1=획득
-- ... 임계 구역 ...
SELECT RELEASE_LOCK('stock:42');

-- PostgreSQL advisory lock (트랜잭션/세션 범위)
SELECT pg_advisory_lock(42);            -- 세션 락
SELECT pg_advisory_xact_lock(42);       -- 트랜잭션 끝나면 자동 해제 (권장)
```
- 장점: 추가 인프라 불필요, 트랜잭션과 자연스럽게 결합(advisory xact lock은 commit/rollback 시 자동 해제).
- 단점: DB 커넥션을 락 보유 동안 점유 → [[Connection-Pool]] 고갈 위험. 락 트래픽이 많으면 DB가 병목. 고빈도 락엔 Redis가 유리.

## 4. Redis 분산 락
가장 널리 쓰이는 방식. 빠르고 TTL로 데드락을 방지.

```bash
# SET NX + TTL — 원자적 획득. 키가 없을 때만 set, 30초 후 자동 만료
SET lock:stock:42 <unique-token> NX PX 30000
```
- **TTL 필수**: 락 보유 클라이언트가 죽어도 만료되어 자동 해제(데드락 방지). 단 TTL이 임계 구역보다 짧으면 작업 중 락이 풀려 위험 → **watchdog**으로 해결.
- **해제는 자기 락만**: 값에 unique token을 넣고, 삭제 시 토큰 일치를 Lua로 원자 검증(만료 후 남의 락을 지우는 사고 방지).

### Redisson RLock — 실무 표준
```java
RLock lock = redissonClient.getLock("stock:42");
try {
    // waitTime 5s 대기, leaseTime 미지정 → watchdog가 자동 연장
    if (lock.tryLock(5, TimeUnit.SECONDS)) {
        stockService.decrease(42L);     // 임계 구역
    }
} finally {
    if (lock.isHeldByCurrentThread()) lock.unlock();
}
```
- **watchdog(자동 갱신)**: `leaseTime`을 주지 않으면 Redisson이 기본 30초 락을 잡고, 작업이 끝나지 않으면 **10초마다 TTL을 갱신**. 임계 구역이 길어도 락이 중간에 풀리지 않는다. 단, 클라이언트가 죽으면 갱신이 멈춰 30초 뒤 해제.

## 5. Redlock 알고리즘과 논란
- **Redlock**: 단일 Redis가 죽으면 락이 사라지는 문제를 막으려고, **N개의 독립 Redis 노드 과반(N/2+1)**에서 락을 획득해야 성공으로 보는 알고리즘.
- **Kleppmann의 비판**: ① 안전성이 시스템 클록·GC 일시정지에 의존(타이밍 가정 위반 시 깨짐), ② 프로세스가 stop-the-world GC로 멈춘 사이 TTL이 만료되면 **두 클라이언트가 동시에 락 보유** 가능.
- 결론: Redis 락은 **상호 배제(효율성)** 목적엔 충분하지만, **절대적 정합성(correctness)**이 필요하면 부족하다. 정합성은 락이 아니라 **fencing token + 자원 측 검증**이나 DB 트랜잭션으로 보장해야 한다.

## 6. 락 없이 해결하기 (우선 검토)
| 기법 | 핵심 |
|------|------|
| **원자적 연산** | `UPDATE stock SET qty=qty-1 WHERE id=? AND qty>0` — DB가 직렬화, 락 불필요 |
| **멱등성** | 같은 요청이 중복돼도 결과 동일하게(요청 ID 기반) → [[Inbox-Pattern]] |
| **낙관적 락 재시도** | 충돌 드물 때 `@Version` + `@Retryable` |
| **메시지 단일 컨슈머** | 키 기준 파티셔닝([[../../spring/messaging/Kafka]])으로 동일 키를 한 컨슈머가 순차 처리 → 락 자체가 불필요 |
| **상태 머신/Saga** | 분산 트랜잭션을 락 대신 보상으로 → [[Saga-Pattern]] |

> [!tip]
> "재고 차감" 한 줄이면 분산 락보다 원자적 `UPDATE ... WHERE qty>0`가 더 빠르고 안전하다. 락은 **여러 자원에 걸친 복합 작업**처럼 원자 연산으로 못 푸는 경우에만.

## 7. 안티패턴
| 안티패턴 | 결과 |
|----------|------|
| TTL 없는 락 | 클라이언트 사망 시 **영구 데드락** |
| 락 누수 | `finally`에서 해제 안 함 → TTL까지 점유 |
| 남의 락 해제 | 토큰 검증 없이 `DEL` → 만료 후 타인 락 삭제 |
| split-brain 무시 | 네트워크 분단으로 양쪽이 락 보유 |
| 임계 구역에 외부 호출 | 느린 I/O로 락 점유 장기화 → 처리량 붕괴 |

## 8. 사용 시 주의
- **락 획득 타임아웃 필수**: 무한 대기 금지(`tryLock(waitTime, ...)`). 못 얻으면 빠른 실패 또는 재시도 큐.
- **임계 구역 최소화**: 락 안에서는 꼭 필요한 작업만. 외부 API·무거운 연산은 락 밖으로.
- **TTL > 최대 작업 시간** 또는 watchdog 사용. TTL이 짧으면 작업 도중 해제 위험.
- 정합성이 돈/재고처럼 치명적이면 Redis 락만 믿지 말고 **DB 제약/원자 연산을 최종 방어선**으로 둔다.

## 9. 관련
- [[Redis]] · [[JPA-Hibernate]] · [[Inbox-Pattern]] · [[Saga-Pattern]] · [[../../spring/messaging/Kafka]] · [[Connection-Pool]]
- [[Transaction-Management]] — Lost Update 방지(낙관 @Version / 비관 FOR UPDATE)와 트랜잭션 경계
