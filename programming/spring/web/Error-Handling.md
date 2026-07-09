---
tags:
  - spring
  - error-handling
  - rest
  - rfc7807
  - problem-details
created: 2026-06-17
---

# 에러 처리 & Problem Details (RFC 7807)

> [!summary] 한 줄 요약
> API 에러 응답은 **일관된 기계 판독 가능 포맷**이어야 한다. **RFC 7807 Problem Details**(`type/title/status/detail/instance`)가 표준이며, Spring 6의 `ProblemDetail` + `@ControllerAdvice`로 예외를 변환한다. 예외 계층 설계·에러 코드 체계·i18n·민감정보 마스킹으로 클라이언트 친화적이고 안전한 응답을 만든다.

---

## 1. 일관된 에러 응답의 중요성
- 엔드포인트마다 에러 형태가 다르면 클라이언트는 **분기 처리 지옥**에 빠진다.
- 일관된 포맷의 이점:
  - 클라이언트가 한 가지 파서로 모든 에러 처리.
  - 에러 코드로 **국제화·자동 분기·재시도 판단** 가능.
  - 로깅/모니터링/알림에서 에러 분류 일관성.
- 안티패턴: `200 OK` + body에 에러 표시, 스택트레이스 그대로 노출, 매번 다른 필드명.

---

## 2. RFC 7807 Problem Details 구조
HTTP API 에러를 위한 표준 미디어 타입 `application/problem+json`.

| 필드 | 의미 | 예 |
|------|------|-----|
| `type` | 에러 유형 식별 URI (문서 링크) | `https://api.example.com/errors/out-of-stock` |
| `title` | 사람이 읽는 요약 (불변) | `"재고 부족"` |
| `status` | HTTP 상태 코드 | `409` |
| `detail` | 이번 발생에 대한 구체 설명 | `"상품 ABC 재고가 3개 남았습니다"` |
| `instance` | 이 에러가 발생한 리소스 URI | `/orders/12345` |

```json
{
  "type": "https://api.example.com/errors/out-of-stock",
  "title": "재고 부족",
  "status": 409,
  "detail": "상품 ABC 재고가 부족합니다 (요청 10, 가용 3)",
  "instance": "/orders/12345",
  "errorCode": "ORDER-4001",        // 확장 필드 (커스텀 가능)
  "timestamp": "2026-06-17T10:00:00Z"
}
```

- `type`/`instance`는 URI지만 반드시 접근 가능할 필요는 없음(식별자 역할).
- 확장 필드를 자유롭게 추가할 수 있어 `errorCode` 같은 사내 규약을 얹기 좋다.

---

## 3. Spring 6 ProblemDetail & @ControllerAdvice
Spring 6 / Boot 3부터 `ProblemDetail`이 1급 지원된다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 비즈니스 예외 → 4xx
    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusiness(BusinessException e) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
                e.getErrorCode().getStatus(), e.getMessage());
        pd.setType(URI.create("https://api.example.com/errors/"
                + e.getErrorCode().name().toLowerCase()));
        pd.setTitle(e.getErrorCode().getTitle());
        pd.setProperty("errorCode", e.getErrorCode().getCode());   // 확장 필드
        pd.setProperty("timestamp", Instant.now());
        return pd;
    }

    // 예상 못한 시스템 예외 → 500 (내부 정보 숨김)
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception e) {
        log.error("처리되지 않은 예외", e);   // 상세는 로그에만
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "일시적인 오류가 발생했습니다");
        pd.setProperty("errorCode", "SYS-5000");
        return pd;     // 스택트레이스/내부 메시지 노출 금지
    }
}
```

- `@RestControllerAdvice`로 전역 예외를 한 곳에서 변환 → 컨트롤러는 비즈니스에 집중.
- `application.yml`에서 `spring.mvc.problemdetails.enabled=true`로 기본 예외도 ProblemDetail로 자동 변환.

---

## 4. 예외 계층 설계
**비즈니스 예외 vs 시스템 예외**를 명확히 구분한다.

```
RuntimeException
 ├── BusinessException (추상)         ← 4xx, 예측된 도메인 규칙 위반
 │    ├── OrderNotFoundException      (404)
 │    ├── OutOfStockException         (409)
 │    └── InvalidCouponException      (400)
 └── (시스템 예외)                     ← 5xx, 예측 못한 인프라/버그
      DB 장애, NPE, 외부 API 타임아웃 등
```

```java
@Getter
public abstract class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;
    protected BusinessException(ErrorCode errorCode, Object... args) {
        super(errorCode.format(args));
        this.errorCode = errorCode;
    }
}
```

- **비즈니스 예외**: 클라이언트에게 의미 있는 메시지·에러코드 제공(4xx).
- **시스템 예외**: 내부 상세를 숨기고 일반 메시지(5xx) + 추적용 로그/traceId.

---

## 5. 검증 에러 변환 (@Valid)
`@Valid` 실패 시 `MethodArgumentNotValidException`이 발생 → 필드별 에러로 변환.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ProblemDetail handleValidation(MethodArgumentNotValidException e) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("입력값 검증 실패");
    pd.setProperty("errorCode", "VALID-4000");
    pd.setProperty("errors", e.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of(
                    "field", fe.getField(),
                    "message", fe.getDefaultMessage(),   // i18n 메시지 키 해석됨
                    "rejected", String.valueOf(fe.getRejectedValue())))
            .toList());
    return pd;
}
```

```json
{
  "title": "입력값 검증 실패", "status": 400, "errorCode": "VALID-4000",
  "errors": [
    { "field": "email", "message": "올바른 이메일 형식이 아닙니다", "rejected": "abc" },
    { "field": "quantity", "message": "1 이상이어야 합니다", "rejected": "0" }
  ]
}
```

- [[DTO]]에 `@NotNull`, `@Email`, `@Min` 등 제약을 선언 → 컨트롤러 진입 전 검증.
- `@ModelAttribute` 바인딩 실패는 `BindException`, 쿼리 파라미터 제약 위반은 `ConstraintViolationException`으로 별도 처리.

---

## 6. 에러 코드 체계
사내 표준 에러 코드로 분류·검색·국제화의 키를 삼는다.

```
[코드 규칙 예: {도메인}-{HTTP대역}{일련}]
  ORDER-4001   주문 - 재고 부족
  ORDER-4040   주문 - 미존재
  AUTH-4010    인증 - 토큰 만료
  SYS-5000     시스템 - 내부 오류
```

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    OUT_OF_STOCK("ORDER-4001", HttpStatus.CONFLICT, "error.order.out_of_stock"),
    ORDER_NOT_FOUND("ORDER-4040", HttpStatus.NOT_FOUND, "error.order.not_found"),
    TOKEN_EXPIRED("AUTH-4010", HttpStatus.UNAUTHORIZED, "error.auth.token_expired");

    private final String code;
    private final HttpStatus status;
    private final String messageKey;   // i18n 메시지 번들 키
}
```

---

## 7. 국제화(i18n)
- 에러 메시지를 코드에 하드코딩하지 말고 **메시지 번들**(`messages_ko.properties`, `messages_en.properties`)로 분리.
- `Accept-Language` 헤더에 따라 `MessageSource`가 적절한 언어로 해석.

```properties
# messages_ko.properties
error.order.out_of_stock=상품 재고가 부족합니다
# messages_en.properties
error.order.out_of_stock=The product is out of stock
```

```java
String message = messageSource.getMessage(
        errorCode.getMessageKey(), args, LocaleContextHolder.getLocale());
```

---

## 8. 민감정보 노출 금지
응답에 절대 포함하면 안 되는 것:

- **스택트레이스·예외 클래스명·SQL** → 공격자에게 내부 구조 노출.
- DB 제약 위반 원문(테이블/컬럼명), 내부 IP/호스트명, 파일 경로.
- 인증 실패 시 "아이디 없음"과 "비밀번호 틀림"을 구분 → 계정 열거 공격. **동일 메시지**로 응답.

```java
// 나쁜 예: e.getMessage()가 "ERROR: duplicate key ... users_email_key" 노출
// 좋은 예: 일반화된 메시지 + traceId로 내부 추적
pd.setProperty("traceId", MDC.get("traceId"));   // 상세는 로그에서 traceId로 조회
```

[[Spring-Cloud-Gateway]] 등 게이트웨이 레벨에서도 백엔드의 5xx 원문이 새지 않도록 에러 응답을 표준화한다.

---

## 9. 클라이언트 친화 응답
- HTTP 상태 코드를 **정확히** 사용([[REST-API]] 규칙) → 클라이언트가 상태코드만으로 1차 분기.
- `errorCode`로 세분화된 분기(같은 400이라도 코드로 구분).
- 재시도 가능 여부 힌트: `503` + `Retry-After`, 멱등 안내.
- 메시지는 **사용자에게 그대로 노출 가능한 수준**으로(개발자용 상세는 별도 필드/로그).

---

## 10. 관련
- [[REST-API]] · [[DTO]] · [[Spring-Cloud-Gateway]]
