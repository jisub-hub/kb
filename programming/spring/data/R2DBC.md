---
tags:
  - database
  - r2dbc
  - reactive
  - async
  - webflux
created: 2026-06-15
---

# R2DBC (Reactive Relational Database Connectivity)

> [!summary] 한 줄 요약
> 관계형 DB를 위한 **논블로킹(non-blocking) 리액티브** 접근 사양. [[JDBC]]가 블로킹이라 리액티브 스택(Spring WebFlux)에서 쓸 수 없는 문제를 해결한다. 결과를 `Mono`/`Flux`(Reactive Streams)로 다룬다.

---

## 1. 왜 등장했나
- JDBC는 **블로킹** → 쿼리 동안 스레드가 묶임. WebFlux(논블로킹) 스택과 맞지 않는다.
- R2DBC는 **적은 스레드로 많은 동시 연결**을 처리(이벤트 루프) → I/O 바운드 고동시성에 유리.
- 지원 드라이버: PostgreSQL, MySQL, MariaDB, MSSQL, H2 등 (Oracle은 별도 r2dbc-oracle).

> [!warning]
> R2DBC는 **JPA가 아니다.** ORM·지연로딩·영속성 컨텍스트가 없고, 매핑은 단순하다(Spring Data JDBC와 유사한 수준 + 리액티브).

---

## 2. 의존성 & 설정
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
implementation 'org.postgresql:r2dbc-postgresql'   // 드라이버
```
```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb     # jdbc: 가 아니라 r2dbc:
    username: app
    password: secret
    pool:
      initial-size: 10
      max-size: 30
```

## 3. 엔티티 & Reactive Repository
```java
@Table("member")
public record Member(@Id Long id, String email, String name) {}

public interface MemberRepository extends ReactiveCrudRepository<Member, Long> {
    Mono<Member> findByEmail(String email);          // 단건 → Mono
    Flux<Member> findByNameContaining(String kw);    // 다건 → Flux

    @Query("SELECT * FROM member WHERE id = :id")
    Mono<Member> findOne(Long id);
}
```

## 4. 서비스 — 리액티브 체이닝 (블로킹 금지!)
```java
@Service
@RequiredArgsConstructor
public class MemberService {
    private final MemberRepository repository;

    public Mono<Member> register(String email, String name) {
        return repository.findByEmail(email)
            .flatMap(existing -> Mono.<Member>error(new DuplicateException(email)))
            .switchIfEmpty(repository.save(new Member(null, email, name)));
        // ⚠️ .block() 호출 금지 — 이벤트 루프 스레드를 막아 전체 성능이 무너진다
    }
}
```

## 5. R2dbcEntityTemplate (저수준 제어)
```java
template.select(Member.class)
        .matching(query(where("name").like("%kim%")))
        .all();                                       // Flux<Member>
```

## 6. 트랜잭션
```java
@Transactional                       // ReactiveTransactionManager 사용
public Mono<Void> transfer(...) { ... }   // 반환이 Mono/Flux여야 트랜잭션이 리액티브 체인에 적용
```

## 7. 장점 / 단점
### ✅ 장점
- **논블로킹** → 적은 스레드로 높은 동시성(I/O 바운드).
- WebFlux와 **엔드투엔드 리액티브** 구성 가능.
- 백프레셔(backpressure) 지원.

### ❌ 단점
- **ORM 아님**: 연관관계·지연로딩·더티체킹 없음 → 매핑·조인 직접 처리.
- **학습 곡선**(리액티브 프로그래밍), 디버깅·스택트레이스 난해.
- 생태계가 JPA보다 얕음. 잘못 쓰면(블로킹 혼입) 오히려 더 느림.

## 8. 언제 쓸 것인가
> [!success] 적합
> - 이미 **WebFlux 리액티브 스택**이고 DB까지 논블로킹이 필요한 경우
> - 매우 높은 동시 접속 + I/O 바운드(스트리밍, 게이트웨이 등)

> [!failure] 부적합
> - 전통적 MVC(블로킹) 스택, 복잡한 도메인/연관관계 → [[JPA-Hibernate]]
> - 팀이 리액티브에 익숙하지 않고 생산성이 우선일 때
>
> 동기 스택에서 단순히 "빠르게" 하려는 목적이면 R2DBC가 답이 아닐 때가 많다 → [[Sync-vs-Async-DataAccess]] 참고.

## 9. 관련
- [[Sync-vs-Async-DataAccess]] · [[JDBC]] · [[JPA-Hibernate]] · [[Connection-Pool]]
