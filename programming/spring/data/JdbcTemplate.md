---
tags:
  - database
  - jdbc
  - spring
  - sync
created: 2026-06-15
---

# JdbcTemplate / Spring Data JDBC

> [!summary] 한 줄 요약
> **JdbcTemplate**: Spring이 제공하는 JDBC 보일러플레이트 제거 도구(커넥션·예외·매핑 자동화). **Spring Data JDBC**: 그 위에서 단순 ORM 매핑을 제공하되, JPA처럼 무겁지 않은 경량 매퍼.

---

## 1. JdbcTemplate

[[JDBC]]의 반복 코드(커넥션 관리, 예외 변환, ResultSet 순회)를 제거한다.

```java
@Repository
@RequiredArgsConstructor
public class MemberDao {
    private final JdbcTemplate jdbcTemplate;

    // 조회 - RowMapper로 매핑
    public Member findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT id, email, name FROM member WHERE id = ?",
            (rs, rowNum) -> new Member(rs.getLong("id"),
                                       rs.getString("email"),
                                       rs.getString("name")),
            id);
    }

    public List<Member> findAll() {
        return jdbcTemplate.query("SELECT id, email, name FROM member",
            new BeanPropertyRowMapper<>(Member.class));   // 컬럼명→필드 자동 매핑
    }

    // 변경
    public int insert(Member m) {
        return jdbcTemplate.update(
            "INSERT INTO member(email, name) VALUES (?, ?)", m.email(), m.name());
    }
}
```

### NamedParameterJdbcTemplate — 이름 기반 파라미터
```java
String sql = "SELECT * FROM member WHERE email = :email";
var params = new MapSqlParameterSource("email", email);
namedTemplate.query(sql, params, rowMapper);
```

---

## 2. Spring Data JDBC

JPA보다 단순한 **경량 ORM**. 객체↔테이블을 매핑하되 **지연로딩/영속성 컨텍스트/더티체킹이 없다** → 동작이 예측 가능하고 가볍다.

```java
@Table("member")
public record Member(@Id Long id, String email, String name) {}

public interface MemberRepository extends CrudRepository<Member, Long> {
    Optional<Member> findByEmail(String email);

    @Query("SELECT * FROM member WHERE name LIKE :kw")
    List<Member> searchByName(String kw);
}
```

### JPA와의 차이
| | Spring Data JDBC | [[JPA-Hibernate]] |
|---|------------------|------|
| 지연 로딩 | ❌ 없음 | ✅ 있음 |
| 영속성 컨텍스트/더티체킹 | ❌ 없음 | ✅ 있음 |
| 복잡도 | 낮음 (예측 가능) | 높음 |
| 애그리거트 매핑 | DDD 애그리거트 친화 | 연관관계 풍부 |

> [!tip]
> "JPA는 과한데 JdbcTemplate은 너무 저수준" 일 때 Spring Data JDBC가 좋은 중간 선택. DDD 애그리거트 저장에도 잘 맞는다.

## 3. 관련
- [[JDBC]] · [[MyBatis]] · [[JPA-Hibernate]] · [[Repository-Pattern]]
