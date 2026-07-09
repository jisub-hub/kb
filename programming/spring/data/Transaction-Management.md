---
tags:
  - spring
  - transaction
  - jpa
  - database
  - concurrency
created: 2026-06-19
---

# @Transactional 전파·격리 (트랜잭션 관리)

> [!summary] 한 줄 요약
> `@Transactional`은 **AOP 프록시**가 메서드 앞뒤로 `begin/commit/rollback`을 끼워 넣는 것. 실무 사고는 거의 다 ① **전파(propagation)** 오해, ② **격리(isolation)** 이상현상, ③ **rollback 규칙**(checked 예외 무시), ④ **프록시 자기호출** 함정에서 나온다.

---

## 1. 동작 원리 — 프록시가 트랜잭션을 건다

```
호출자 → [프록시] → 실제 빈
          │
          ├ begin (PlatformTransactionManager.getTransaction)
          ├ 실제 메서드 실행
          └ commit (정상) / rollback (예외)
```
- 트랜잭션 = **스레드에 바인딩된 커넥션** 한 개 위에서 진행(`TransactionSynchronizationManager`가 ThreadLocal로 보관).
- 같은 트랜잭션 안의 모든 JPA/JDBC 작업은 **같은 물리 커넥션**을 쓴다 → 전파가 "커넥션을 새로 쓰느냐"를 결정한다.

> [!warning] ⚠️ 자기 호출(self-invocation)은 프록시를 못 거친다
> ```java
> public void outer() { inner(); }          // ← this.inner() = 프록시 우회
> @Transactional public void inner() { ... } // 트랜잭션 안 걸림!
> ```
> 같은 빈 내부에서 `this.메서드()`로 부르면 AOP 프록시를 거치지 않아 `@Transactional`이 **무시된다**. → 별도 빈으로 분리하거나 self-injection. ([[Spring-Core]] AOP 프록시 원리)

---

## 2. 전파 (Propagation) — 7종

| 전파 옵션 | 기존 트랜잭션 있을 때 | 없을 때 | 용도 |
|----------|---------------------|--------|------|
| **REQUIRED** (기본) | 참여(join) | 새로 시작 | 대부분의 경우 |
| **REQUIRES_NEW** | **중단(suspend)+새 트랜잭션** | 새로 시작 | 독립 커밋(로그·감사) |
| **NESTED** | savepoint 생성 | 새로 시작 | 부분 롤백 |
| **SUPPORTS** | 참여 | 트랜잭션 없이 실행 | 읽기 유연 |
| **NOT_SUPPORTED** | 중단하고 비트랜잭션 실행 | 비트랜잭션 | 트랜잭션 불필요 구간 |
| **MANDATORY** | 참여 | **예외** | 반드시 상위 트랜잭션 요구 |
| **NEVER** | **예외** | 비트랜잭션 | 트랜잭션 금지 강제 |

### 2.1 REQUIRED — 하나의 운명 공동체
```
outer(REQUIRED) → inner(REQUIRED)  = 같은 트랜잭션
  inner에서 예외 → 전체 롤백
  inner를 try-catch로 잡아도 → 트랜잭션은 이미 rollback-only 마킹됨
     → outer 커밋 시도 시 UnexpectedRollbackException ⚠️
```
> 흔한 함정: "안쪽 예외를 잡았으니 바깥은 커밋되겠지" → ❌. REQUIRED는 물리 트랜잭션이 하나라 **rollback-only 플래그**가 공유된다.

### 2.2 REQUIRES_NEW — 독립 커밋, 그러나 커넥션 2개
```
outer(REQUIRED) ──suspend──► inner(REQUIRES_NEW)
   커넥션 #1 (대기)            커넥션 #2 (별도 begin/commit)
   inner 커밋/롤백은 outer와 무관 → 감사로그·실패기록에 적합
```
> [!danger] REQUIRES_NEW 커넥션 풀 고갈 (실장애 단골)
> outer가 커넥션 #1을 **쥔 채** inner가 #2를 풀에서 또 꺼낸다. 루프 안에서 REQUIRES_NEW를 호출하거나 풀이 작으면 → 한 요청이 2개씩 점유 → **데드락성 풀 고갈**. → [[Connection-Pool]] 사이징에 REQUIRES_NEW 사용을 반영해야 함.

### 2.3 NESTED — savepoint 부분 롤백
```
JDBC savepoint 기반 → inner 실패 시 savepoint까지만 롤백, outer는 계속
  ⚠️ JPA/Hibernate + 일부 드라이버에서 제약(savepoint 미지원 조합 존재)
  ⚠️ REQUIRES_NEW와 달리 outer가 롤백되면 NESTED도 함께 사라짐(독립 아님)
```

---

## 3. 격리 수준 (Isolation) — 동시성 이상현상

| 수준 | Dirty Read | Non-repeatable | Phantom | 비고 |
|------|:---:|:---:|:---:|------|
| READ_UNCOMMITTED | ⭕ | ⭕ | ⭕ | 거의 미사용 |
| **READ_COMMITTED** | ❌ | ⭕ | ⭕ | **PostgreSQL 기본** |
| REPEATABLE_READ | ❌ | ❌ | ⭕(MVCC선 사실상❌) | **MySQL(InnoDB) 기본** |
| SERIALIZABLE | ❌ | ❌ | ❌ | 직렬화, 비용 큼 |

```
- Dirty Read       : 커밋 안 된 데이터를 읽음
- Non-repeatable   : 같은 행을 두 번 읽었는데 값이 다름(중간 UPDATE 커밋)
- Phantom          : 같은 조건 재조회 시 행 수가 달라짐(중간 INSERT)
- Lost Update      : 두 트랜잭션이 read-modify-write → 한쪽 갱신 소실
   (격리만으론 못 막음 → 비관 락 SELECT FOR UPDATE / 낙관 락 @Version)
```
> [!tip] DB마다 기본값이 다르다
> PostgreSQL=READ_COMMITTED, MySQL InnoDB=REPEATABLE_READ. `@Transactional(isolation=...)`로 올리기 전에, **Lost Update는 격리가 아니라 락([[Distributed-Lock]] 낙관/비관)으로** 막는 게 정석.

---

## 4. Rollback 규칙 — checked 예외는 기본 롤백 안 됨

```java
@Transactional
public void pay() throws IOException {
    repo.save(...);
    throw new IOException();   // ⚠️ checked 예외 → 기본적으로 커밋됨(롤백 X)!
}
```
- 기본 규칙: **RuntimeException·Error → 롤백**, **checked Exception → 커밋**.
- 이유: 스프링은 checked 예외를 "비즈니스적으로 복구 가능한 상황"으로 간주(EJB 관행 계승).
- 강제하려면:
```java
@Transactional(rollbackFor = Exception.class)        // checked 포함 전부 롤백
@Transactional(noRollbackFor = BusinessException.class) // 특정 예외는 커밋 유지
```

---

## 5. readOnly 최적화

```java
@Transactional(readOnly = true)
public List<Order> findAll() { ... }
```
- Hibernate flush 모드를 **MANUAL**로 → 더티 체킹/flush 생략(성능·메모리 이득).
- 일부 드라이버/프록시는 readOnly 힌트를 **읽기 복제본 라우팅**에 사용(쓰기는 primary, 읽기는 replica).
- 단순 조회 서비스 메서드엔 거의 항상 `readOnly = true` 권장.

---

## 6. 트랜잭션 안에서 하면 안 되는 것 ⚠️

```
트랜잭션 = 커넥션을 점유한 상태. 그 안에서 느린 작업을 하면 커넥션이 묶인다.

금지/주의:
  ① 외부 API 호출(HTTP)을 트랜잭션 안에서 → 응답 지연만큼 커넥션 점유 → 풀 고갈
  ② 대용량 파일 I/O·메일 발송 등 느린 부수효과
  ③ 트랜잭션 커밋 "후"에 이벤트를 발행해야 하는데 안에서 발행
     → @TransactionalEventListener(phase = AFTER_COMMIT) 사용
  ④ 메시지 발행과 DB 변경의 원자성 → 2PC 대신 [[Outbox-Pattern]]
```
> 원칙: **트랜잭션은 짧게, DB 작업만.** 외부 호출·메시징은 트랜잭션 경계 밖 또는 커밋 후로.

---

## 7. 체크리스트

```
□ 조회 메서드에 readOnly = true 붙였나
□ checked 예외인데 롤백돼야 하면 rollbackFor 지정했나
□ 같은 빈 내부 호출로 @Transactional이 무시되고 있지 않나(self-invocation)
□ REQUIRES_NEW가 커넥션을 2개 점유함을 풀 사이징에 반영했나
□ 트랜잭션 안에서 외부 API/느린 I/O를 호출하고 있지 않나
□ Lost Update를 격리로 막으려 하고 있지 않나(→ 락/@Version)
□ 커밋 후 후처리는 AFTER_COMMIT 리스너로 분리했나
```

---

## 8. 관련
- [[Spring-Core]] — AOP 프록시 원리(self-invocation 함정의 근원)
- [[JPA-Hibernate]] — 영속성 컨텍스트·flush·더티체킹(readOnly와 직결)
- [[Connection-Pool]] — REQUIRES_NEW·긴 트랜잭션이 풀 사이징에 주는 영향
- [[Distributed-Lock]] — Lost Update 방지(낙관 @Version / 비관 FOR UPDATE)
- [[Saga-Pattern]] — 분산 환경에서 로컬 트랜잭션을 잇는 보상 트랜잭션
- [[Outbox-Pattern]] — DB 변경과 메시지 발행의 원자성
