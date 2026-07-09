---
tags:
  - testing
  - testcontainers
  - integration-test
  - docker
created: 2026-06-15
---

# Testcontainers

> [!summary] 한 줄 요약
> **테스트 코드 안에서 Docker 컨테이너를 띄우는 라이브러리**. 인메모리 H2 대신 실제 PostgreSQL·MySQL·Redis·Kafka 등을 테스트마다 격리된 컨테이너로 실행해, **운영 환경과 동일한 조건**에서 통합 테스트할 수 있다.

---

## 1. 왜 필요한가

| | H2 인메모리 | Testcontainers |
|---|---|---|
| DB 방언 차이 | ❌ H2 전용 SQL·함수 | ✅ 실제 DB 그대로 |
| 네이티브 타입/인덱스 | ❌ 미지원 많음 | ✅ 완전 지원 |
| 환경 구성 복잡도 | 낮음 | 중간 (Docker 필요) |
| 신뢰도 | 낮음 (가짜 DB) | 높음 (실제 DB) |
| CI 속도 | 빠름 | 약간 느림 (컨테이너 시작) |

> 실무에서 H2로 테스트 통과했는데 운영 MySQL에서 터지는 사례가 많다 → Testcontainers로 해결.

---

## 2. 의존성 (Spring Boot 3.x)

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:postgresql'   // 쓰는 DB 모듈만 추가
    // testImplementation 'org.testcontainers:mysql'
    // testImplementation 'org.testcontainers:kafka'
}
```

> Spring Boot 3.1+는 `spring-boot-testcontainers` 가 BOM으로 버전 관리. 별도 버전 지정 불필요.

---

## 3. 기본 사용법 — `@Testcontainers` + `@Container`

```java
@SpringBootTest
@Testcontainers
class MemberRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void overrideProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired MemberRepository memberRepository;

    @Test
    void 회원_저장_조회() {
        Member saved = memberRepository.save(new Member("test@example.com", "홍길동"));
        assertThat(memberRepository.findById(saved.getId())).isPresent();
    }
}
```

- `static` 컨테이너 → 테스트 클래스 전체에서 **한 번만 시작·종료** (빠름).
- 인스턴스 필드 → 각 `@Test`마다 새 컨테이너 (격리되지만 느림).

---

## 4. Spring Boot 3.1+ 간편 방식 — `@ServiceConnection`

`@DynamicPropertySource` 없이 **자동 연결** 가능.

```java
@SpringBootTest
@Testcontainers
class MemberRepositoryTest {

    @Container
    @ServiceConnection                      // URL/계정 자동 주입
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    // @DynamicPropertySource 불필요
}
```

> `@ServiceConnection`은 컨테이너 타입을 인식해 `spring.datasource.*` 프로퍼티를 자동으로 채워준다. Redis(`RedisContainer`), Kafka(`KafkaContainer`) 등도 동일.

---

## 5. 공통 설정 분리 — `@SpringBootTest` 베이스 클래스

컨테이너를 여러 테스트에서 공유할 때는 추상 베이스 클래스로 뽑는다.

```java
@SpringBootTest
@Testcontainers
abstract class IntegrationTestBase {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true);   // Testcontainers Desktop 사용 시 컨테이너 재사용
}

// 각 테스트는 상속만 받으면 됨
class OrderServiceTest extends IntegrationTestBase {
    @Autowired OrderService orderService;
    ...
}
```

---

## 6. 여러 컨테이너 (DB + Redis + Kafka)

```java
@SpringBootTest
@Testcontainers
abstract class FullStackIntegrationBase {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static RedisContainer redis =
        new RedisContainer("redis:7-alpine");

    @Container
    @ServiceConnection
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));
}
```

---

## 7. 슬라이스 테스트 (`@DataJpaTest`) 와 조합

```java
@DataJpaTest                           // JPA 레이어만 로드 (빠름)
@AutoConfigureTestDatabase(replace = NONE)  // 내장 H2 비활성화
@Testcontainers
class PostRepositorySliceTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired PostRepository postRepository;

    @Test
    void 페이징_조회() {
        // 실제 PostgreSQL로 JPA 레포지토리만 검증
    }
}
```

> `@AutoConfigureTestDatabase(replace = NONE)` 이 없으면 Spring이 자동으로 H2로 교체해버린다.

---

## 8. CI (GitHub Actions) 설정

Docker-in-Docker 없이 **runner에 Docker가 있으면 바로 동작**한다.

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest   # Docker 기본 탑재
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }
      - run: ./gradlew test
```

컨테이너 이미지 Pull 시간을 줄이려면 GitHub Actions 캐시나 **Testcontainers Cloud** 활용.

---

## 9. 컨테이너 모듈 치트시트

| 기술 | 모듈 | 컨테이너 클래스 |
|---|---|---|
| PostgreSQL | `org.testcontainers:postgresql` | `PostgreSQLContainer` |
| MySQL | `org.testcontainers:mysql` | `MySQLContainer` |
| Redis | `org.testcontainers:redis` | `RedisContainer` |
| Kafka | `org.testcontainers:kafka` | `KafkaContainer` |
| MongoDB | `org.testcontainers:mongodb` | `MongoDBContainer` |
| Elasticsearch | `org.testcontainers:elasticsearch` | `ElasticsearchContainer` |
| 임의 컨테이너 | — | `GenericContainer` |

---

## 10. Singleton Container 패턴

### 문제: 테스트 클래스마다 컨테이너 재시작

```
기본 @Container static 방식:
  TestClassA 시작 → PostgreSQL 컨테이너 start (3~5초)
  TestClassA 종료 → PostgreSQL 컨테이너 stop
  TestClassB 시작 → PostgreSQL 컨테이너 start (3~5초 또 기다림)
  TestClassB 종료 → PostgreSQL 컨테이너 stop
  ...
  → 테스트 클래스 10개 = 컨테이너 10번 재시작

Singleton 방식:
  JVM 시작 → PostgreSQL 컨테이너 start (딱 1번)
  TestClassA, TestClassB, ... 전부 같은 컨테이너 사용
  JVM 종료 → PostgreSQL 컨테이너 stop (딱 1번)
  → 빌드 시간 대폭 절감
```

### Singleton Container 클래스

```java
// src/test/java/com/example/support/TestContainerSingleton.java
/**
 * JVM 당 컨테이너를 단 한 번만 기동하는 Singleton.
 * static initializer로 최초 접근 시점에 컨테이너를 시작하고,
 * JVM 종료 시 ShutdownHook으로 자동 정리된다.
 */
public abstract class TestContainerSingleton {

    // PostgreSQL — static final + static block = JVM 당 1회 기동
    public static final PostgreSQLContainer<?> POSTGRES;

    // Redis
    public static final GenericContainer<?> REDIS;

    // Kafka (선택)
    public static final KafkaContainer KAFKA;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);           // ← Testcontainers Desktop 사용 시 JVM 재시작해도 재사용

        REDIS = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379)
            .withReuse(true);

        KAFKA = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
            .withReuse(true);

        // 병렬 시작으로 총 기동 시간 단축
        Startables.deepStart(POSTGRES, REDIS, KAFKA).join();
    }
}
```

> [!tip] `withReuse(true)` 활성화 조건
> `~/.testcontainers.properties` 에 `testcontainers.reuse.enable=true` 추가 필요.
> CI에서는 reuse를 사용하지 않는 것이 안전 (상태 오염 방지).

---

### Abstract Base — 모든 통합 테스트가 상속

```java
// src/test/java/com/example/support/AbstractIntegrationTest.java

/**
 * 모든 통합 테스트의 공통 베이스.
 * - Singleton 컨테이너(PostgreSQL / Redis / Kafka)를 한 번만 기동
 * - @DynamicPropertySource로 컨테이너 접속 정보를 Spring에 주입
 * - @ActiveProfiles("test") 적용
 * - 테스트 간 DB 격리를 위한 @Transactional 제공
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Transactional                              // 각 테스트 후 롤백 (DB 상태 초기화)
public abstract class AbstractIntegrationTest extends TestContainerSingleton {

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        // PostgreSQL
        registry.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);

        // Redis
        registry.add("spring.data.redis.host", REDIS::getHost);
        registry.add("spring.data.redis.port", () -> REDIS.getMappedPort(6379));

        // Kafka
        registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
    }

    // 서브클래스에서 공통 픽스처·헬퍼를 추가로 제공할 수 있음
}
```

### 슬라이스 테스트용 Abstract Base

```java
// JPA 슬라이스 전용 (전체 컨텍스트 불필요 → 빠름)
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ActiveProfiles("test")
public abstract class AbstractRepositoryTest extends TestContainerSingleton {

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

---

### 구체 테스트 클래스 — 상속만 하면 끝

```java
// ── 통합 테스트 ────────────────────────────────────────────────
class OrderServiceIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("주문 생성 후 DB에 저장되는지 확인")
    void placeOrder_SavesToDatabase() {
        var result = orderService.place(new OrderCommand("item-1", 2));

        assertThat(orderRepository.findById(result.orderId())).isPresent();
    }

    // @Transactional 상속 → 각 테스트 종료 시 자동 롤백
    // → 테스트 간 데이터 격리 자동 보장
}

// ── Repository 슬라이스 테스트 ─────────────────────────────────
class MemberRepositoryTest extends AbstractRepositoryTest {

    @Autowired
    private MemberRepository memberRepository;

    @Test
    void findByEmail_ReturnsCorrectMember() {
        memberRepository.save(new Member("hong@example.com", "홍길동"));

        var found = memberRepository.findByEmail("hong@example.com");

        assertThat(found).isPresent()
            .get().extracting(Member::getName).isEqualTo("홍길동");
    }
}

// ── Security 포함 통합 테스트 ──────────────────────────────────
class OrderApiIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private JwtUtil jwtUtil;

    @Test
    @WithMockJwtUser(userId = 1L, email = "test@example.com")
    void myOrders_RequiresAuthentication() {
        // AbstractIntegrationTest 상속 → 컨테이너 재사용
        // @WithMockJwtUser → SecurityUtil.getCurrentUserId() 동작
        var resp = restTemplate.getForEntity("/api/orders/me", String.class);
        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

### 전체 클래스 계층 구조

```
TestContainerSingleton          ← static 블록으로 컨테이너 기동 (JVM 당 1회)
    │
    ├── AbstractIntegrationTest     ← @SpringBootTest + @DynamicPropertySource
    │       │                          + @Transactional (롤백)
    │       ├── OrderServiceIntegrationTest
    │       ├── PaymentServiceIntegrationTest
    │       └── OrderApiIntegrationTest
    │
    └── AbstractRepositoryTest      ← @DataJpaTest + @DynamicPropertySource
            │
            ├── MemberRepositoryTest
            └── OrderRepositoryTest

(별도 계층)
    AbstractWebMvcTest              ← @WebMvcTest + @Import(TestSecurityConfig)
            │                          (DB 컨테이너 불필요 — MockBean 사용)
            ├── OrderControllerTest
            └── MemberControllerTest
```

---

### @Transactional 주의사항

```java
// ❌ 문제: @Transactional + RANDOM_PORT 조합
// TestRestTemplate은 별도 HTTP 클라이언트 스레드를 사용 → 테스트 트랜잭션 참여 불가
// → restTemplate.getForEntity() 결과가 빈 목록으로 보일 수 있음

@SpringBootTest(webEnvironment = RANDOM_PORT)
@Transactional
class BadTest extends AbstractIntegrationTest {
    @Autowired TestRestTemplate restTemplate;

    @Test
    void test() {
        memberRepository.save(...); // 이 트랜잭션은 HTTP 요청 스레드에 안 보임
        restTemplate.getForEntity("/api/members", ...); // 항상 빈 결과
    }
}

// ✅ 해결책 1: RANDOM_PORT 테스트는 @Transactional 제거 + @Sql로 픽스처 삽입
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Sql("/sql/member-fixture.sql")         // 테스트 전 데이터 삽입
@Sql(scripts = "/sql/cleanup.sql",      // 테스트 후 정리
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class MemberApiTest extends AbstractIntegrationTest { ... }

// ✅ 해결책 2: MOCK 환경 + MockMvc는 @Transactional 정상 동작
@SpringBootTest(webEnvironment = MOCK)
@Transactional
class MemberApiMockTest extends AbstractIntegrationTest {
    @Autowired MockMvc mockMvc;
    ...
}
```

---

## 11. 관련
- [[Unit-Integration-Testing]] — @TestConfiguration, SecurityUtil, @WithMockJwtUser
- [[Load-Testing]] · [[../data/JPA-Hibernate]]
