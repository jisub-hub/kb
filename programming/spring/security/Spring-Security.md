---
tags:
  - security
  - spring-security
  - authentication
  - authorization
created: 2026-06-15
---

# Spring Security

> [!summary] 한 줄 요약
> Spring의 **인증/인가 프레임워크**. 서블릿 **필터 체인**으로 모든 요청을 가로채 인증·권한·CSRF·CORS 등을 처리한다. 선언적 설정 + 메서드 보안.

---

## 1. 핵심 구조
```
요청 → [SecurityFilterChain]
        ├─ SecurityContextPersistenceFilter   (SecurityContext 로딩)
        ├─ 인증 필터 (UsernamePassword / BearerToken ...)
        ├─ AuthorizationFilter                 (권한 검사)
        └─ ExceptionTranslationFilter          (401/403 처리)
      → Controller
```
- **Authentication**: 인증 주체 정보(`principal`, `authorities`).
- **SecurityContext**: 현재 인증 정보 보관(ThreadLocal).
- **GrantedAuthority**: 권한(`ROLE_ADMIN` 등).
- **UserDetailsService**: 사용자 조회 인터페이스.

---

## 2. 설정 (Spring Security 6, Lambda DSL)
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity            // @PreAuthorize 활성화
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                       // 무상태 API면 보통 off
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/login").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/orders/**").hasAuthority("ORDER_WRITE")
                .anyRequest().authenticated())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults())); // JWT 검증
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();                     // 비밀번호 해싱
    }
}
```

## 3. 사용자 인증 (DB 기반)
```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {
    private final MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        Member m = memberRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));
        return User.withUsername(m.getEmail())
                   .password(m.getPassword())                   // 이미 인코딩된 값
                   .authorities(m.getRoles().toArray(String[]::new))
                   .build();
    }
}
```

## 4. 메서드 보안 (인가)
```java
@Service
public class OrderService {
    @PreAuthorize("hasRole('ADMIN')")                           // 메서드 진입 전 권한 검사
    public void deleteAll() { ... }

    @PreAuthorize("#userId == authentication.name")             // 본인만
    public Order getMyOrder(String userId) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.name") // 반환 후 검사
    public Order get(Long id) { ... }
}
```

## 5. 현재 사용자 접근
```java
@GetMapping("/me")
public String me(@AuthenticationPrincipal UserDetails user) {
    return user.getUsername();
}
// 또는
var auth = SecurityContextHolder.getContext().getAuthentication();
```

## 6. WebFlux(리액티브) 보안
```java
@Bean
SecurityWebFilterChain springSecurity(ServerHttpSecurity http) {
    return http.authorizeExchange(e -> e.anyExchange().authenticated())
               .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
               .build();
}
```
> [[MVC-vs-WebFlux]] 에 따라 `HttpSecurity` vs `ServerHttpSecurity` 사용.

## 7. 베스트 프랙티스
- 비밀번호는 **BCrypt/Argon2** 해싱(평문 저장 금지).
- API는 보통 **STATELESS + 토큰**([[OAuth2-JWT]]).
- 최소 권한 원칙, 권한은 코드가 아닌 **데이터/정책**으로 관리.
- CORS/CSRF 정책 명확히(브라우저 세션이면 CSRF on, 토큰 API면 off).

## 8. 관련
- [[OAuth2-JWT]] · [[Spring-Cloud-Gateway]] · [[REST-API]] · [[MSA]]
