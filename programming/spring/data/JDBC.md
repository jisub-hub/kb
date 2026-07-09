---
tags:
  - database
  - jdbc
  - sync
  - java
created: 2026-06-15
---

# JDBC (Java Database Connectivity)

> [!summary] 한 줄 요약
> 자바에서 관계형 DB에 접속·쿼리하기 위한 **표준 저수준 API**. MyBatis·JPA·JdbcTemplate 등 거의 모든 동기 데이터 접근 기술이 내부적으로 JDBC 위에서 동작한다.

---

## 1. 특징
- **동기/블로킹**: 쿼리가 끝날 때까지 스레드가 대기.
- 가장 **저수준** → 세밀한 제어 가능하지만 **보일러플레이트가 많다**.
- 핵심 객체: `DataSource` → `Connection` → `PreparedStatement` → `ResultSet`.

## 2. 순수 JDBC 예시 (보일러플레이트의 실체)
```java
public Member findById(Long id) {
    String sql = "SELECT id, email, name FROM member WHERE id = ?";
    try (Connection conn = dataSource.getConnection();
         PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setLong(1, id);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                return new Member(rs.getLong("id"),
                                  rs.getString("email"),
                                  rs.getString("name"));
            }
            return null;
        }
    } catch (SQLException e) {
        throw new RuntimeException(e);   // 매번 예외/자원관리 반복
    }
}
```
> 커넥션 획득·해제, 예외 처리, ResultSet 매핑을 **매번 직접** 해야 한다 → 그래서 [[JdbcTemplate]]/MyBatis/JPA가 등장.

## 3. 트랜잭션
```java
conn.setAutoCommit(false);
try {
    // ... 여러 SQL ...
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```
> Spring에서는 `@Transactional` 이 이 과정을 대신 처리한다.

## 4. 언제 직접 쓰나
- 거의 직접 쓸 일은 없다. **추상화 계층(JdbcTemplate/MyBatis/JPA)** 을 쓰는 게 정석.
- 극단적 성능/제어가 필요한 일부 배치·드라이버 레벨 작업 정도.

## 5. 관련
- [[JdbcTemplate]] · [[MyBatis]] · [[JPA-Hibernate]] · [[Connection-Pool]]
- 비동기 대응: [[R2DBC]]
