---
tags:
  - security
  - oauth2
  - jwt
  - oidc
  - token
created: 2026-06-15
---

# OAuth2 / OIDC / JWT

> [!summary] 한 줄 요약
> **OAuth2**: 권한 위임(액세스 토큰 발급) 프로토콜. **OIDC**: OAuth2 위에 인증(ID 토큰)을 얹은 표준. **JWT**: 서명된 자가포함 토큰 포맷. MSA에서 무상태 인증의 핵심.

---

## 1. 용어 정리
| 용어 | 설명 |
|------|------|
| **Resource Owner** | 사용자(자원 주인) |
| **Client** | 자원에 접근하려는 앱 |
| **Authorization Server** | 토큰 발급(예: Keycloak, Auth0, Cognito) |
| **Resource Server** | 보호된 API(토큰 검증) |
| **Access Token** | 자원 접근용(보통 짧은 수명) |
| **Refresh Token** | 액세스 토큰 재발급용(긴 수명) |
| **ID Token (OIDC)** | 사용자 인증 정보(JWT) |

## 2. 대표 Grant Type
- **Authorization Code (+ PKCE)**: 웹/모바일 표준. 가장 안전.
- **Client Credentials**: 서버-서버(사용자 없음, M2M). [[MSA]] 서비스 간.
- ~~Implicit / Password~~: 보안상 폐기/비권장.

```
[Authorization Code]
User → Client → Auth Server(로그인/동의) → code → Client
Client → Auth Server (code+secret) → Access/ID/Refresh Token
Client → Resource Server (Bearer Access Token) → 자원
```

---

## 3. JWT 구조
```
header.payload.signature   (Base64URL, 점으로 구분)
```
```json
// payload (claims)
{ "sub": "user-123", "roles": ["USER"], "exp": 1718450000, "iss": "https://auth.example.com" }
```
- **서명(signature)** 으로 위변조 검증 → 서버가 상태 저장 없이 검증 가능(**무상태**).
- ⚠️ payload는 **암호화 아님**(Base64 디코딩되면 보임) → 민감정보 넣지 말 것.

---

## 4. Resource Server (JWT 검증) — Spring
```groovy
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
```
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/myrealm   # JWKS 자동 발견
```
```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(jwt ->
            jwt.jwtAuthenticationConverter(converter())));       // claim → 권한 매핑
    return http.build();
}

@Bean
JwtAuthenticationConverter converter() {
    var roles = new JwtGrantedAuthoritiesConverter();
    roles.setAuthoritiesClaimName("roles");
    roles.setAuthorityPrefix("ROLE_");
    var conv = new JwtAuthenticationConverter();
    conv.setJwtGrantedAuthoritiesConverter(roles);
    return conv;
}
```
```java
// 토큰 클레임 접근
@GetMapping("/me")
public String me(@AuthenticationPrincipal Jwt jwt) {
    return jwt.getSubject();           // sub claim
}
```

## 5. Client (로그인 위임) — Spring
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ...
            client-secret: ...
            scope: openid,profile,email
```
```java
http.oauth2Login(Customizer.withDefaults());   // OIDC 로그인 + 세션
```

## 6. MSA에서의 흐름
```
Client → [Gateway] (토큰 검증/주입) → Service A → Service B
                                         (서비스 간엔 Client Credentials 또는 토큰 전파)
```
- 게이트웨이에서 1차 검증, 각 서비스도 토큰 재검증(zero-trust). → [[Spring-Cloud-Gateway]]

## 7. 베스트 프랙티스
- **Access Token 짧게**(분 단위) + Refresh Token으로 갱신.
- 토큰은 **HTTPS**로만, 저장은 HttpOnly 쿠키(웹) 권장(XSS 대비).
- JWT 폐기(로그아웃) 어려움 → 짧은 수명 + 블랙리스트/짧은 갱신주기로 보완, 또는 opaque 토큰.
- 검증: 서명(`iss`, `aud`, `exp`) 필수. JWKS로 키 롤링 대응.

## 8. 관련
- [[Spring-Security]] · [[Spring-Cloud-Gateway]] · [[MSA]] · [[REST-API]]
