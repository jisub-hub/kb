---
tags:
  - database
  - mybatis
  - sql-mapper
  - sync
created: 2026-06-15
---

# MyBatis

> [!summary] 한 줄 요약
> **SQL Mapper 프레임워크**. SQL을 개발자가 **직접 작성**하고, 그 결과를 객체에 매핑해준다. ORM(JPA)과 달리 SQL을 완전히 통제할 수 있어 복잡한 쿼리·튜닝·레거시 DB에 강하다.

---

## 1. 특징
- **동기/블로킹**, [[JDBC]] 기반.
- SQL을 XML 또는 애너테이션으로 직접 작성 → **SQL 가시성·제어력 최고**.
- ORM이 아니므로 객체 그래프 자동 관리·더티체킹·지연로딩 **없음**.
- 한국 SI/금융권에서 특히 널리 사용.

---

## 2. Mapper 인터페이스 + XML
```java
// Mapper 인터페이스
@Mapper
public interface MemberMapper {
    Member findById(Long id);
    List<Member> search(MemberSearchCond cond);
    int insert(Member member);
}
```
```xml
<!-- MemberMapper.xml -->
<mapper namespace="com.example.MemberMapper">

    <select id="findById" resultType="Member">
        SELECT id, email, name FROM member WHERE id = #{id}
    </select>

    <!-- 동적 SQL - MyBatis의 강점 -->
    <select id="search" resultType="Member">
        SELECT * FROM member
        <where>
            <if test="email != null"> AND email = #{email} </if>
            <if test="name  != null"> AND name LIKE CONCAT('%', #{name}, '%') </if>
        </where>
        ORDER BY id DESC
    </select>

    <insert id="insert" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO member(email, name) VALUES (#{email}, #{name})
    </insert>
</mapper>
```

### `#{}` vs `${}`
- `#{}` → **PreparedStatement 파라미터 바인딩** (SQL 인젝션 안전). **기본 사용.**
- `${}` → 문자열 그대로 치환 (정렬 컬럼명 등 제한적 용도, 인젝션 위험).

---

## 3. 애너테이션 방식 (간단한 쿼리)
```java
@Mapper
public interface MemberMapper {
    @Select("SELECT id, email, name FROM member WHERE id = #{id}")
    Member findById(Long id);

    @Insert("INSERT INTO member(email, name) VALUES (#{email}, #{name})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insert(Member m);
}
```

## 4. Spring Boot 연동
```groovy
// build.gradle
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
```
```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.domain
  configuration:
    map-underscore-to-camel-case: true   # user_name → userName 자동 매핑
```

## 5. 장점 / 단점
### ✅ 장점
- SQL **완전 제어** → 복잡한 조인/통계/튜닝에 유리.
- 동적 SQL 강력 (`<if>`, `<foreach>`, `<choose>`).
- 레거시/비표준 스키마 대응 쉬움. 학습 곡선 완만.

### ❌ 단점
- SQL을 직접 다 써야 함 → **반복 작업, DB 종속**.
- 객체 그래프/연관관계 자동 관리 없음.
- DB 벤더 변경 시 SQL 수정 부담.

> [!tip] JPA vs MyBatis
> **도메인 중심·생산성** → [[JPA-Hibernate]]. **SQL 중심·튜닝·복잡쿼리** → MyBatis. 실무에선 둘을 **혼용**(기본 CRUD는 JPA, 복잡 통계는 MyBatis)하기도 한다.

## 6. MyBatis Dynamic SQL (DSL)

MyBatis 공식 **타입 세이프 쿼리 빌더**. XML 없이 Java 코드로 SQL을 조립한다. [[QueryDSL]]이 JPA 전용인 것과 달리 순수 SQL 기반이라 MyBatis와 함께 사용 가능.

### 의존성
```groovy
implementation 'org.mybatis.dynamic-sql:mybatis-dynamic-sql:1.5.2'
```

### 테이블/컬럼 메타 정의
```java
public class MemberDynamicSqlSupport {
    public static final SqlTable member = SqlTable.of("member");
    public static final SqlColumn<Long>   id    = member.column("id",    JDBCType.BIGINT);
    public static final SqlColumn<String> email = member.column("email", JDBCType.VARCHAR);
    public static final SqlColumn<String> name  = member.column("name",  JDBCType.VARCHAR);
}
```

### Mapper 인터페이스 (CommonCountMapper 등 믹스인 활용)
```java
@Mapper
public interface MemberMapper extends CommonInsertMapper<Member>,
                                       CommonCountMapper {

    @SelectProvider(type = SqlProviderAdapter.class, method = "select")
    @Results(id = "MemberResult", value = {
        @Result(column = "id",    property = "id",    id = true),
        @Result(column = "email", property = "email"),
        @Result(column = "name",  property = "name")
    })
    List<Member> selectMany(SelectStatementProvider statement);

    @SelectProvider(type = SqlProviderAdapter.class, method = "select")
    @ResultMap("MemberResult")
    Optional<Member> selectOne(SelectStatementProvider statement);
}
```

### 동적 쿼리 조립
```java
import static com.example.mapper.MemberDynamicSqlSupport.*;
import static org.mybatis.dynamic.sql.SqlBuilder.*;

@RequiredArgsConstructor
@Repository
public class MemberRepository {

    private final MemberMapper mapper;

    public List<Member> search(String emailCond, String nameCond) {
        SelectStatementProvider stmt = select(id, email, name)
            .from(member)
            .where(email, isEqualToWhenPresent(emailCond))   // null이면 자동 제외
            .and(name, isLikeWhenPresent(nameCond)
                        .map(s -> "%" + s + "%"))
            .orderBy(id.descending())
            .build()
            .render(RenderingStrategies.MYBATIS3);

        return mapper.selectMany(stmt);
    }

    public Optional<Member> findById(Long memberId) {
        SelectStatementProvider stmt = select(id, email, name)
            .from(member)
            .where(id, isEqualTo(memberId))
            .build()
            .render(RenderingStrategies.MYBATIS3);

        return mapper.selectOne(stmt);
    }
}
```

> `isEqualToWhenPresent` / `isLikeWhenPresent` 등 `*WhenPresent` 조건은 **null이면 자동으로 조건을 제외**한다.

### JOIN — 같은 테이블을 두 번 조인할 때 (작성자/수정자)

게시글에 `created_by`, `updated_by` 모두 `member` 테이블을 가리키는 경우, **alias별 SqlTable 인스턴스**를 별도로 선언한다.

```java
// PostDynamicSqlSupport.java
public class PostDynamicSqlSupport {
    public static final SqlTable post       = SqlTable.of("post");
    public static final SqlColumn<Long>   id        = post.column("id",         JDBCType.BIGINT);
    public static final SqlColumn<String> title     = post.column("title",      JDBCType.VARCHAR);
    public static final SqlColumn<Long>   createdBy = post.column("created_by", JDBCType.BIGINT);
    public static final SqlColumn<Long>   updatedBy = post.column("updated_by", JDBCType.BIGINT);
}

// MemberAliasSqlSupport.java — alias 전용 (같은 member 테이블, 두 가지 역할)
public class MemberAliasSqlSupport {
    // creator alias
    public static final SqlTable creatorTable   = SqlTable.of("member").withAlias("creator");
    public static final SqlColumn<Long>   creatorId   = creatorTable.column("id",   JDBCType.BIGINT);
    public static final SqlColumn<String> creatorName = creatorTable.column("name", JDBCType.VARCHAR);

    // updater alias
    public static final SqlTable updaterTable   = SqlTable.of("member").withAlias("updater");
    public static final SqlColumn<Long>   updaterId   = updaterTable.column("id",   JDBCType.BIGINT);
    public static final SqlColumn<String> updaterName = updaterTable.column("name", JDBCType.VARCHAR);
}
```

```java
// DTO
public record PostDetailDto(
    Long   postId,
    String title,
    String creatorName,
    String updaterName
) {}
```

```java
// Mapper
@Mapper
public interface PostMapper {
    @SelectProvider(type = SqlProviderAdapter.class, method = "select")
    @Results(id = "PostDetailResult", value = {
        @Result(column = "id",           property = "postId"),
        @Result(column = "title",        property = "title"),
        @Result(column = "creator_name", property = "creatorName"),
        @Result(column = "updater_name", property = "updaterName")
    })
    List<PostDetailDto> selectPostDetails(SelectStatementProvider stmt);
}
```

```java
// Repository — JOIN 쿼리 조립
import static com.example.mapper.PostDynamicSqlSupport.*;
import static com.example.mapper.MemberAliasSqlSupport.*;
import static org.mybatis.dynamic.sql.SqlBuilder.*;

@RequiredArgsConstructor
@Repository
public class PostRepository {

    private final PostMapper mapper;

    public List<PostDetailDto> findAllWithUsers() {
        SelectStatementProvider stmt = select(
                id,
                title,
                creatorName.as("creator_name"),
                updaterName.as("updater_name"))
            .from(post)
            .leftJoin(creatorTable).on(createdBy, equalTo(creatorId))
            .leftJoin(updaterTable).on(updatedBy, equalTo(updaterId))
            .build()
            .render(RenderingStrategies.MYBATIS3);

        return mapper.selectPostDetails(stmt);
    }
}
```

생성되는 SQL:
```sql
SELECT post.id, post.title,
       creator.name AS creator_name,
       updater.name AS updater_name
FROM   post
LEFT JOIN member creator ON post.created_by = creator.id
LEFT JOIN member updater ON post.updated_by = updater.id
```

> [!tip] JOIN이 3개 이상으로 복잡해지면
> Dynamic SQL은 컬럼 alias 관리가 번거로워진다. 이때는 **XML Mapper로 fallback**하는 것이 더 읽기 쉽다. 두 방식을 같은 `@Mapper` 인터페이스에 섞어 써도 된다.

### XML DSL vs Dynamic SQL 비교
|           | XML Mapper    | MyBatis Dynamic SQL |
| --------- | ------------- | ------------------- |
| SQL 작성 방식 | XML `<if>` 태그 | Java 메서드 체이닝        |
| 타입 안전성    | ❌ 문자열         | ✅ 컴파일 검증            |
| IDE 지원    | 보통            | ✅ 자동완성              |
| 학습 곡선     | 낮음            | 중간                  |
| 복잡 SQL    | XML이 더 직관적    | 간단·중간 쿼리에 적합        |

---

## 7. 관련
- [[JDBC]] · [[JPA-Hibernate]] · [[Sync-vs-Async-DataAccess]] · [[QueryDSL]]
