---
tags:
  - spring
  - rest
  - api
  - http
created: 2026-06-15
---

# REST API

> [!summary] 한 줄 요약
> HTTP의 메서드·URI·상태코드를 활용해 **리소스(자원)** 를 표현·조작하는 아키텍처 스타일. 사람이 읽기 쉽고 범용적이라 공개 API·웹/모바일 백엔드의 사실상 표준.

---

## 1. REST 원칙 (Richardson Maturity Model)
- **Level 0**: HTTP를 단순 터널로 사용(RPC over HTTP).
- **Level 1**: 리소스 분리(URI로 자원 식별).
- **Level 2**: HTTP 메서드/상태코드 의미 활용 ⭐ (대부분 여기).
- **Level 3**: HATEOAS(응답에 다음 행동 링크 포함).

### 핵심 규칙
- **자원은 명사**, 행위는 **HTTP 메서드**로.
  - `GET /orders/123` (O) / `GET /getOrder?id=123` (X)
- 복수형 컬렉션: `/orders`, `/orders/{id}`, `/orders/{id}/items`

| 메서드 | 용도 | 멱등성 | 안전 |
|--------|------|--------|------|
| GET | 조회 | ✅ | ✅ |
| POST | 생성 | ❌ | ❌ |
| PUT | 전체 수정/대체 | ✅ | ❌ |
| PATCH | 부분 수정 | ❌(일반적) | ❌ |
| DELETE | 삭제 | ✅ | ❌ |

### 주요 상태 코드
- `200 OK`, `201 Created`(+Location), `204 No Content`
- `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`
- `500 Internal Server Error`, `503 Service Unavailable`

---

## 2. Spring MVC REST 컨트롤러
```java
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    @GetMapping("/{id}")
    public OrderDto get(@PathVariable Long id) {
        return orderService.findById(id);
    }

    @GetMapping
    public Page<OrderDto> list(@RequestParam(required = false) String status,
                               Pageable pageable) {                 // 페이징
        return orderService.search(status, pageable);
    }

    @PostMapping
    public ResponseEntity<Void> create(@RequestBody @Valid CreateOrderRequest req) {
        Long id = orderService.create(req);
        return ResponseEntity.created(URI.create("/orders/" + id)).build();  // 201 + Location
    }

    @PatchMapping("/{id}")
    public OrderDto update(@PathVariable Long id, @RequestBody UpdateOrderRequest req) {
        return orderService.update(id, req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)                          // 204
    public void delete(@PathVariable Long id) { orderService.delete(id); }
}
```

## 3. 예외 처리 (일관된 에러 응답)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ProblemDetail> notFound(EntityNotFoundException e) {
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
        return ResponseEntity.status(404).body(pd);    // RFC 7807 ProblemDetail
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail validation(MethodArgumentNotValidException e) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setDetail(e.getBindingResult().getAllErrors().toString());
        return pd;
    }
}
```

## 4. 호출 측 (클라이언트)
```java
// RestClient (Spring 6.1+, 동기, 권장)
RestClient client = RestClient.create("http://order-service");
OrderDto order = client.get().uri("/orders/{id}", id)
                       .retrieve().body(OrderDto.class);

// WebClient (논블로킹, WebFlux) — [[MVC-vs-WebFlux]]
Mono<OrderDto> mono = webClient.get().uri("/orders/{id}", id)
                               .retrieve().bodyToMono(OrderDto.class);

// OpenFeign (선언적, MSA에서 흔함)
@FeignClient(name = "order-service")
interface OrderClient {
    @GetMapping("/orders/{id}") OrderDto get(@PathVariable Long id);
}
```

## 5. 베스트 프랙티스
- **DTO로 입출력**(엔티티 직접 노출 금지), 검증(`@Valid`).
- **버저닝**: `/v1/orders` 또는 헤더 기반.
- **페이징/필터/정렬** 표준화, 일관된 에러 포맷(ProblemDetail/RFC 7807).
- HATEOAS는 필요 시에만(대부분 Level 2로 충분).
- 멱등성: PUT/DELETE는 멱등, POST 재시도엔 Idempotency-Key 고려(→ 6절).

## 6. 멱등성 & 재시도 안전성

> 네트워크는 at-least-once다 — 응답이 유실되면 클라이언트가 재시도한다. **같은 요청이 두 번 도착해도 결과가 같아야** 이중 결제·중복 생성을 막는다.

### 6.1 메서드별 멱등성

| 메서드 | 멱등 | 안전(safe) | 비고 |
|--------|------|-----------|------|
| GET | ✅ | ✅ | 조회, 부수효과 없음 |
| PUT | ✅ | ❌ | 전체 대체 — 몇 번 해도 동일 상태 |
| DELETE | ✅ | ❌ | 삭제 — 두 번째는 이미 없음(멱등) |
| **POST** | ❌ | ❌ | 생성 — 재시도 시 **중복 생성 위험** |
| PATCH | △ | ❌ | 구현에 따라 (절대값 SET은 멱등, 증분은 비멱등) |

```
멱등 ≠ 안전:
  멱등 = 여러 번 호출해도 "결과 상태"가 같음
  안전 = 서버 상태를 안 바꿈 (GET만)
주의: DELETE는 멱등이지만 응답 코드는 다를 수 있음(200 vs 404) — 상태는 동일
```

### 6.2 POST 멱등화 — Idempotency-Key 패턴

POST(결제·주문 생성)는 본질적으로 비멱등이라, **클라이언트가 멱등키를 부여**해 서버가 중복을 거른다. (Stripe 등 결제 API 표준)

```
1. 클라이언트: 요청마다 고유 키 생성 (UUID), 재시도 시 같은 키 재사용
   POST /payments
   Idempotency-Key: 7b3f...c9

2. 서버: 키를 저장소(DB/Redis)에 기록
   - 처음 보는 키 → 처리 + (키, 결과) 저장
   - 이미 본 키 → 처리 안 하고 저장된 결과 반환 (같은 응답)
```

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResult> pay(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody PaymentRequest req) {
    // 멱등키로 기존 결과 조회 (Redis/DB) — 있으면 그대로 반환
    return idempotencyStore.find(key)
        .map(saved -> ResponseEntity.ok(saved))
        .orElseGet(() -> {
            PaymentResult r = paymentService.charge(req);
            idempotencyStore.save(key, r, Duration.ofHours(24)); // TTL
            return ResponseEntity.ok(r);
        });
}
```

> 키 저장과 비즈니스 처리는 **같은 트랜잭션**이어야 원자적이다(처리 후 저장 전 크래시 → 재처리). 구현 정석은 [[Inbox-Pattern]](수신 멱등성)과 동일하다. 동시 같은 키 경쟁은 [[Distributed-Lock]] 또는 unique 제약으로.

### 6.3 클라이언트 재시도 규칙

```
멱등 메서드(GET/PUT/DELETE) → 안전하게 재시도 가능 (exponential backoff)
POST                        → Idempotency-Key 있을 때만 재시도
재시도 대상: 네트워크 타임아웃·5xx·429(Retry-After)
재시도 금지: 4xx(요청 자체 오류) — 고쳐서 재전송
```

> 서버 멱등 + 클라이언트 안전 재시도가 짝을 이뤄야 한다. → 메시지 큐 맥락의 멱등은 [[MQ-Concepts]], [[Inbox-Pattern]].

## 7. REST vs gRPC
- 공개 API·브라우저·범용·디버깅 용이 → **REST**
- 서비스 간 고성능·스트리밍·강타입 계약 → [[gRPC]]

## 8. 관련
- [[gRPC]] · [[MVC-vs-WebFlux]] · [[Spring-Cloud-Gateway]] · [[MSA]]
- [[Inbox-Pattern]] · [[Distributed-Lock]] · [[Error-Handling]] (멱등·재시도 연계)
