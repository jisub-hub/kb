---
tags:
  - testing
  - spring
  - junit5
  - mockito
  - tdd
created: 2026-06-16
---

# 단위·통합 테스트 — Spring Boot

> [!summary] 한 줄 요약
> **테스트 피라미드**: 단위(70%) → 통합(20%) → E2E(10%). Spring Boot는 슬라이스 테스트(`@WebMvcTest`, `@DataJpaTest`)로 통합 테스트를 빠르게, Testcontainers로 실 DB와 연동한다.

---

## 1. 테스트 피라미드

```
        △
       /E2E\       10% — 느림, 비용↑, Selenium/Playwright
      /──────\
     / 통합  \     20% — 슬라이스·Testcontainers
    /──────────\
   /  단위 테스트 \ 70% — 빠름, 격리됨, JUnit5 + Mockito
  /──────────────\
```

---

## 2. 단위 테스트 — JUnit 5 + Mockito

```java
// 서비스 단위 테스트: 외부 의존성을 Mock으로 격리
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentClient paymentClient;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("재고 부족 시 주문 실패 예외 발생")
    void placeFails_WhenOutOfStock() {
        // given
        var cmd = new OrderCommand("item-1", 100);
        given(orderRepository.findStock("item-1")).willReturn(5);   // Stub

        // when & then
        assertThatThrownBy(() -> orderService.place(cmd))
            .isInstanceOf(OutOfStockException.class)
            .hasMessageContaining("재고 부족");
    }

    @Test
    @DisplayName("주문 성공 시 이벤트 발행")
    void placeSuccess_PublishesEvent() {
        // given
        given(orderRepository.findStock("item-1")).willReturn(50);
        given(paymentClient.charge(any())).willReturn(PaymentResult.success("pay-123"));

        // when
        orderService.place(new OrderCommand("item-1", 1));

        // then
        verify(paymentClient, times(1)).charge(any());   // 검증
        verify(orderRepository, never()).rollback(any());
    }

    @ParameterizedTest
    @CsvSource({ "0", "-1", "-100" })
    @DisplayName("수량 0 이하 주문 거부")
    void rejectInvalidQuantity(int qty) {
        assertThatThrownBy(() -> orderService.place(new OrderCommand("item-1", qty)))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

### Mock vs Stub vs Spy

| | 역할 | 언제 |
|--|------|------|
| **Mock** | 호출 검증 (`verify`) | 상호작용 테스트 |
| **Stub** | 리턴값 설정 (`given/when`) | 상태 기반 테스트 |
| **Spy** | 실제 객체 + 부분 Mock | 레거시 코드 통합 시 |

---

## 3. Controller 슬라이스 테스트 — @WebMvcTest

```java
// Spring MVC 레이어만 로드 (Service, Repository는 Mock)
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean                          // Spring 컨텍스트에 Mock 등록
    private OrderService orderService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("POST /orders — 정상 주문 201 반환")
    void createOrder_Returns201() throws Exception {
        // given
        var req = new OrderRequest("item-1", 2);
        given(orderService.place(any())).willReturn(new OrderResult("order-123"));

        // when & then
        mockMvc.perform(post("/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.orderId").value("order-123"))
            .andDo(print());   // 요청/응답 콘솔 출력
    }

    @Test
    @DisplayName("수량 없이 요청 시 400 Bad Request")
    void createOrder_Returns400_WhenQuantityMissing() throws Exception {
        mockMvc.perform(post("/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"itemId\": \"item-1\"}"))  // quantity 누락
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[0].field").value("quantity"));
    }
}
```

---

## 4. Repository 슬라이스 테스트 — @DataJpaTest

```java
// JPA + H2 인메모리 DB (혹은 @Testcontainers로 실 PG)
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)   // 실 DB 사용 시 (Testcontainers)
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    @DisplayName("상태별 주문 조회 쿼리 검증")
    void findByStatus() {
        // given
        entityManager.persistAndFlush(Order.of("member-1", OrderStatus.PENDING));
        entityManager.persistAndFlush(Order.of("member-1", OrderStatus.COMPLETED));

        // when
        var pending = orderRepository.findByStatus(OrderStatus.PENDING);

        // then
        assertThat(pending).hasSize(1);
        assertThat(pending.get(0).getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```

---

## 5. 통합 테스트 — @SpringBootTest

```java
// 전체 컨텍스트 로드 — 느리므로 핵심 흐름만
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
class OrderIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("주문 → 결제 → 재고 감소 전체 플로우")
    void orderFlow_EndToEnd() {
        var req = new OrderRequest("item-1", 1);

        var resp = restTemplate.postForEntity("/orders", req, OrderResult.class);

        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(resp.getBody().getOrderId()).isNotNull();
    }
}
```

---

## 6. 픽스처 & 테스트 데이터

```java
// 오브젝트 마더 패턴 — 테스트 데이터 팩토리
public class OrderFixture {
    public static Order pendingOrder() {
        return Order.builder()
            .memberId("test-member")
            .status(OrderStatus.PENDING)
            .totalAmount(Money.of(10_000))
            .createdAt(Instant.now())
            .build();
    }

    public static Order completedOrder() {
        return pendingOrder().complete("pay-123");
    }
}

// 사용: given(repo.findById(1L)).willReturn(Optional.of(OrderFixture.pendingOrder()));
```

---

## 7. 테스트 설정 분리

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
  jpa:
    hibernate:
      ddl-auto: create-drop     # 테스트마다 스키마 재생성
  kafka:
    bootstrap-servers: ${spring.embedded.kafka.brokers}  # 임베디드 Kafka

logging:
  level:
    org.hibernate.SQL: DEBUG    # 테스트 중 쿼리 확인
    org.hibernate.type: TRACE   # 바인딩 파라미터 확인
```

---

## 8. @TestConfiguration — 테스트 전용 Bean 재정의

`@TestConfiguration`은 테스트 컨텍스트에서만 활성화되는 Bean 설정 클래스다. 본 설정(`@Configuration`)을 건드리지 않고 Mock·Fake·Stub 구현체로 교체할 때 사용한다.

```java
// ── 선언 위치 1: 테스트 클래스 내부 (해당 테스트 클래스에만 적용) ──
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @TestConfiguration          // ← 이 테스트 클래스 스코프
    static class TestConfig {

        @Bean
        @Primary                // 동일 타입이 있으면 이 Bean이 우선
        EmailSender fakeSender() {
            return message -> log.info("[FAKE] Skipping email: {}", message);
        }
    }

    @Autowired MockMvc mockMvc;
    @MockBean  OrderService orderService;
    ...
}
```

```java
// ── 선언 위치 2: 별도 파일 (여러 테스트에서 공유) ──
// src/test/java/com/example/config/TestSecurityConfig.java
@TestConfiguration
public class TestSecurityConfig {

    /**
     * 테스트에서 Security Filter Chain을 단순화.
     * CSRF 비활성화 + 모든 요청 permitAll → MockMvc 인증 우회 가능.
     */
    @Bean
    @Order(1)
    SecurityFilterChain testSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .build();
    }
}

// 사용하는 테스트 클래스에서 @Import로 명시적 로드
@WebMvcTest(OrderController.class)
@Import(TestSecurityConfig.class)
class OrderControllerWithAuthTest { ... }
```

```java
// ── 선언 위치 3: @SpringBootTest + 외부 서비스 Fake ──
@SpringBootTest
@Import(TestExternalConfig.class)
class PaymentIntegrationTest { ... }

@TestConfiguration
class TestExternalConfig {

    @Bean
    @Primary
    PaymentGateway fakePaymentGateway() {
        // 실제 PG사 연동 대신 항상 성공하는 Fake
        return request -> PaymentResult.success("FAKE-" + UUID.randomUUID());
    }

    @Bean
    @Primary
    SmsClient fakeSmsClient() {
        return (to, msg) -> log.debug("[FAKE SMS] to={} msg={}", to, msg);
    }
}
```

### @TestConfiguration vs @MockBean 선택

| | `@TestConfiguration` | `@MockBean` |
|--|---------------------|------------|
| 구현 | 실제 Fake/Stub 구현체 | Mockito 프록시 |
| 동작 | 정의한 로직 그대로 실행 | 기본 null 반환 (`given()` 필요) |
| 상태 유지 | 가능 (인스턴스 변수) | 매 테스트마다 리셋 |
| 적합 | 복잡한 행동 Fake, 공유 설정 | 단순 호출 검증, 리턴값 설정 |

---

## 9. SecurityUtil — 로그인 정보 공통 유틸

### 프로덕션 구현

```java
/**
 * SecurityContextHolder에서 인증 정보를 꺼내는 유틸.
 * 컨트롤러·서비스 어디서나 static 메서드로 현재 로그인 사용자 접근.
 */
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class SecurityUtil {

    /** 현재 로그인 사용자 ID (Long). 미인증 시 예외. */
    public static Long getCurrentUserId() {
        return getPrincipal().getUserId();
    }

    /** 현재 로그인 사용자 이메일. */
    public static String getCurrentUserEmail() {
        return getPrincipal().getEmail();
    }

    /** 현재 로그인 사용자 Role 목록. */
    public static Set<String> getCurrentUserRoles() {
        return getAuthentication().getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toUnmodifiableSet());
    }

    /** 인증 여부 확인. */
    public static boolean isAuthenticated() {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null && auth.isAuthenticated()
            && !(auth instanceof AnonymousAuthenticationToken);
    }

    // ── 내부 헬퍼 ──────────────────────────────────────────────

    private static Authentication getAuthentication() {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            throw new AuthenticationCredentialsNotFoundException("인증 정보 없음");
        }
        return auth;
    }

    private static UserPrincipal getPrincipal() {
        var auth = getAuthentication();
        if (auth.getPrincipal() instanceof UserPrincipal principal) {
            return principal;
        }
        throw new IllegalStateException("Principal 타입 불일치: " + auth.getPrincipal().getClass());
    }
}
```

### JWT 파싱 유틸

```java
@Component
@RequiredArgsConstructor
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    private SecretKey key;

    @PostConstruct
    private void init() {
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }

    /** 토큰 파싱 → Claims 반환. 만료·변조 시 예외. */
    public Claims parseClaims(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)
            .getBody();
    }

    /** Claims → UserPrincipal 변환. */
    public UserPrincipal toUserPrincipal(Claims claims) {
        return UserPrincipal.builder()
            .userId(claims.get("userId", Long.class))
            .email(claims.getSubject())
            .roles(Set.copyOf(claims.get("roles", List.class)))
            .build();
    }

    /** JwtAuthenticationFilter에서 사용: 토큰 → Authentication 객체. */
    public Authentication toAuthentication(String token) {
        var claims = parseClaims(token);
        var principal = toUserPrincipal(claims);
        return new UsernamePasswordAuthenticationToken(
            principal, token, principal.getAuthorities());
    }

    /** 토큰 생성 (로그인 시). */
    public String generateToken(UserPrincipal principal) {
        return Jwts.builder()
            .setSubject(principal.getEmail())
            .claim("userId", principal.getUserId())
            .claim("roles", principal.getRoles())
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(1, ChronoUnit.HOURS)))
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
    }
}
```

### 서비스·컨트롤러에서 사용

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderResult place(OrderRequest req) {
        // SecurityUtil로 현재 로그인 사용자 ID 조회
        Long memberId = SecurityUtil.getCurrentUserId();

        Order order = Order.of(memberId, req.getItemId(), req.getQuantity());
        return new OrderResult(orderRepository.save(order).getId());
    }

    public List<Order> getMyOrders() {
        // 직접 메서드 주입 없이 유틸 사용
        return orderRepository.findByMemberId(SecurityUtil.getCurrentUserId());
    }
}

// 컨트롤러에서도 동일
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/me")
    public ResponseEntity<List<OrderSummary>> myOrders() {
        // @AuthenticationPrincipal UserPrincipal principal 매개변수 없이도 OK
        Long userId = SecurityUtil.getCurrentUserId();
        ...
    }
}
```

---

### 테스트에서 SecurityUtil 사용

#### 방법 1 — `@WithMockUser` (Role 기반 단순 테스트)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @MockBean OrderService orderService;

    @Test
    @WithMockUser(username = "user@example.com", roles = {"USER"})
    void getMyOrders_Returns200() throws Exception {
        given(orderService.getMyOrders()).willReturn(List.of());

        mockMvc.perform(get("/api/orders/me"))
            .andExpect(status().isOk());
    }

    @Test
    void getMyOrders_Returns401_WhenNotAuthenticated() throws Exception {
        mockMvc.perform(get("/api/orders/me"))
            .andExpect(status().isUnauthorized());
    }
}
```

#### 방법 2 — 커스텀 `@WithMockJwtUser` (UserPrincipal 포함)

`@WithMockUser`는 `UserPrincipal` 타입을 Principal로 세팅하지 못한다. `SecurityUtil.getCurrentUserId()`처럼 커스텀 Principal을 꺼내는 코드는 아래 방식이 필요하다.

```java
// ① 어노테이션 정의
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithMockJwtUserSecurityContextFactory.class)
public @interface WithMockJwtUser {
    long userId() default 1L;
    String email()  default "test@example.com";
    String[] roles() default {"ROLE_USER"};
}

// ② SecurityContext 생성 팩토리
public class WithMockJwtUserSecurityContextFactory
        implements WithSecurityContextFactory<WithMockJwtUser> {

    @Override
    public SecurityContext createSecurityContext(WithMockJwtUser annotation) {
        var principal = UserPrincipal.builder()
            .userId(annotation.userId())
            .email(annotation.email())
            .roles(Set.of(annotation.roles()))
            .build();

        var auth = new UsernamePasswordAuthenticationToken(
            principal, "test-token", principal.getAuthorities());

        var context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(auth);
        return context;
    }
}

// ③ 테스트에서 사용
@WebMvcTest(OrderController.class)
class OrderControllerSecurityTest {

    @MockBean OrderService orderService;

    @Test
    @WithMockJwtUser(userId = 42L, email = "hong@example.com", roles = {"ROLE_USER"})
    void getMyOrders_UsesCurrentUserId() throws Exception {
        // SecurityUtil.getCurrentUserId() → 42L 반환
        given(orderService.getMyOrders()).willReturn(List.of());

        mockMvc.perform(get("/api/orders/me"))
            .andExpect(status().isOk());

        // 서비스가 호출되었는지 검증
        verify(orderService, times(1)).getMyOrders();
    }
}
```

#### 방법 3 — 통합 테스트에서 실제 JWT 토큰 발급

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderIntegrationTest extends AbstractIntegrationTest {

    @Autowired JwtUtil jwtUtil;
    @Autowired TestRestTemplate restTemplate;

    @Test
    void getMyOrders_WithRealJwt() {
        // 테스트용 UserPrincipal로 실제 JWT 생성
        var principal = UserPrincipal.builder()
            .userId(1L).email("test@example.com")
            .roles(Set.of("ROLE_USER")).build();

        String token = jwtUtil.generateToken(principal);

        var headers = new HttpHeaders();
        headers.setBearerAuth(token);

        var resp = restTemplate.exchange(
            "/api/orders/me", HttpMethod.GET,
            new HttpEntity<>(headers), OrderSummary[].class);

        assertThat(resp.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

## 10. 관련
- [[Testcontainers]] — 실 DB 기반 통합 테스트, Singleton 컨테이너 패턴
- [[Load-Testing]] — 성능 테스트
- [[../data/JPA-Hibernate]] · [[../architecture/DDD]]
- [[../security/OAuth2-JWT|JWT]] — JWT 발급·검증 구현
