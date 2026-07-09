---
tags:
  - spring
  - dto
  - web
  - api
  - design
created: 2026-06-15
---

# DTO (Data Transfer Object)

> [!summary] 한 줄 요약
> 계층/시스템 **경계를 넘어 데이터를 전달**하기 위한 전용 객체. 도메인 엔티티를 그대로 외부에 노출하지 않고, **표현(API 응답)·입력(요청)에 맞춘 형태**로 변환해 주고받는다. 핵심 동기는 **도메인과 표현의 분리(decoupling)**.

---

## 1. DTO란 무엇인가
- 비즈니스 로직이 **없는** 순수 데이터 운반 객체(getter/필드 위주, 가급적 불변).
- "**경계(boundary)**"에서 쓰인다: Controller ↔ Client(API), 때로는 Service ↔ Controller, 외부 시스템 연동.
- 도메인 모델(Entity)과 **모양이 달라도 된다** — 화면/응답/입력에 최적화.

```
[Client] ⇄ (Request/Response DTO) ⇄ [Controller] ⇄ [Service] ⇄ [Domain Entity] ⇄ [DB]
                  ↑ 표현 계층 전용                          ↑ 비즈니스 규칙/영속성
```

### 비슷한 것들과 구분
| 객체 | 역할 |
|------|------|
| **Entity** | 도메인 규칙·영속성(JPA @Entity). 식별자·불변식 보유 |
| **DTO** | 경계 전달용. 로직 없음 |
| **VO(Value Object)** | 도메인 내부의 값 개념(Money, Address). 불변·동등성 |
| **Command/Query** | 의도를 표현하는 입력 DTO의 일종 ([[CQRS]]) |

---

## 2. 왜 쓰는가 (도입 이유)
1. **도메인 캡슐화** — 내부 구조(엔티티 필드·연관관계)를 외부에 숨김. 내부 변경이 API 계약을 깨지 않음.
2. **표현 최적화** — 화면/응답마다 필요한 필드만, 합쳐서/가공해서 전달(여러 엔티티 조합).
3. **보안** — 민감 필드(비밀번호, 내부 상태, 권한 플래그)의 **과다 노출 방지**.
4. **계약 안정성** — API 스펙(DTO)과 DB 스키마(Entity)를 **독립적으로 진화**.
5. **입력 검증 지점** — 요청 DTO에 `@Valid` 제약을 모아 표현 계층에서 검증.
6. **직렬화 안전** — JPA 지연로딩/순환참조로 인한 직렬화 사고 회피(4장).

---

## 3. 언제 / 어디서 써야 하나

> [!success] DTO를 쓰는 게 맞는 상황
> - **공개 API(REST/gRPC) 요청·응답** — 거의 항상 권장. → [[REST-API]]
> - 응답이 **여러 엔티티의 조합**이거나 화면 전용 가공 데이터일 때
> - **민감/내부 필드**를 가진 엔티티를 다룰 때
> - API 계약과 DB 스키마의 **수명·변경 주기가 다를 때**
> - [[CQRS]]의 **조회(Query) 측** — read 전용 DTO/Projection으로 반환
> - 외부 시스템/레거시 연동 경계(+ [[DDD]]의 ACL)

> [!failure] DTO가 과한 상황 (실용적 예외)
> - **아주 단순한 내부 CRUD**, 엔티티와 응답이 1:1이고 노출 위험 없는 경우
> - 빠른 프로토타입/PoC (단, 공개 API로 가기 전 반드시 DTO화)
> - 읽기 전용 프로젝션이 충분한 단순 조회(인터페이스 Projection으로 대체 가능)

원칙: **외부로 나가는 경계는 DTO, 내부 깊은 곳은 도메인.** 의심되면 DTO를 두는 쪽이 안전하다.

---

## 4. ⚠️ DTO 없이 엔티티를 직접 쓸 때의 Tradeoff

엔티티를 컨트롤러 응답/요청에 그대로 노출하면 단기적으론 편하지만 아래 비용을 치른다.

| 문제 | 설명 |
|------|------|
| **강결합 (API ↔ DB)** | 엔티티 필드명/구조를 바꾸면 **API 계약이 즉시 깨진다.** 리팩터링이 외부 영향으로 막힘 |
| **과다 노출/보안** | `password`, 내부 상태, 권한 등 **숨겨야 할 필드가 응답에 샘** (`@JsonIgnore` 누락 위험) |
| **지연로딩 직렬화 사고** | JPA LAZY 연관을 트랜잭션 밖에서 직렬화 → `LazyInitializationException`, 또는 의도치 않은 추가 쿼리 폭발 |
| **순환 참조** | 양방향 연관(`Order↔OrderLine`)을 직렬화하면 **무한 루프/StackOverflow** |
| **N+1 유발** | 응답 직렬화가 연관을 줄줄이 로딩 → 성능 저하 ([[JPA-Hibernate]]) |
| **Over/Under-fetching** | 화면이 필요로 하는 데이터와 엔티티 모양이 안 맞아 너무 많이/적게 전달 |
| **입력 바인딩 취약점** | 요청을 엔티티에 직접 바인딩 → **Mass Assignment**(클라이언트가 `isAdmin=true` 같은 필드 주입) 위험 |
| **검증 책임 혼재** | 표현 계층 검증 제약이 도메인 엔티티에 섞여 모델이 오염 |

> [!danger] 가장 흔한 사고
> "엔티티를 그대로 `@RestController`에서 반환" → 비밀번호 노출 + LAZY 직렬화 예외 + 순환참조. **공개 API에서 엔티티 직접 노출은 안티패턴.**

### 반대로, DTO 도입의 비용(없지 않다)
- **매핑 코드/보일러플레이트** 증가 (Entity ↔ DTO 변환).
- 유사 클래스 증가(요청/응답/내부 DTO).
- 매핑 누락·불일치 버그 가능성.
→ 이 비용은 **매핑 도구(MapStruct)·record·프로젝션**으로 상당히 줄일 수 있다(6장). 대부분의 경우 **결합도/보안 이득 > 매핑 비용**.

---

## 5. Spring 구현 예시

### 5.1. 요청/응답 DTO (record + 검증)
```java
// 요청 DTO - 입력 검증을 여기에 모은다
public record CreateOrderRequest(
        @NotNull Long customerId,
        @NotEmpty List<@Valid OrderLineRequest> lines) {
    public record OrderLineRequest(@NotNull Long productId, @Positive int quantity) {}
}

// 응답 DTO - 화면/계약에 맞춘 형태 (엔티티 구조와 무관)
public record OrderResponse(
        Long orderId,
        String customerName,        // 다른 엔티티에서 합쳐옴
        String status,
        long totalAmount,
        List<LineResponse> lines) {
    public record LineResponse(String productName, int quantity, long price) {}
}
```

### 5.2. Controller — 경계에서만 DTO 사용
```java
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<Void> create(@RequestBody @Valid CreateOrderRequest req) {
        Long id = orderService.create(req);                  // 엔티티는 노출되지 않음
        return ResponseEntity.created(URI.create("/orders/" + id)).build();
    }

    @GetMapping("/{id}")
    public OrderResponse get(@PathVariable Long id) {
        return orderService.getResponse(id);                 // DTO만 반환
    }
}
```

### 5.3. 매핑 — 수동
```java
public static OrderResponse from(Order order) {     // 정적 팩토리 매핑
    return new OrderResponse(
        order.getId(),
        order.getCustomer().getName(),
        order.getStatus().name(),
        order.totalAmount().amount(),
        order.getLines().stream()
             .map(l -> new OrderResponse.LineResponse(
                     l.getProductName(), l.getQuantity(), l.getPrice()))
             .toList());
}
```

### 5.4. 매핑 — MapStruct (보일러플레이트 제거)
```java
@Mapper(componentModel = "spring")
public interface OrderMapper {
    @Mapping(target = "customerName", source = "customer.name")
    @Mapping(target = "orderId", source = "id")
    OrderResponse toResponse(Order order);          // 구현은 컴파일 타임에 자동 생성
}
```

### 5.5. 조회 최적화 — DTO 프로젝션 (엔티티를 안 거침)
```java
// JPQL 생성자 표현식 - 필요한 컬럼만 SELECT → over-fetching/N+1 회피
@Query("""
    select new com.example.order.OrderResponse(
        o.id, o.customer.name, o.status, o.totalAmount)
    from Order o where o.id = :id
""")
Optional<OrderResponse> findResponseById(Long id);

// 또는 인터페이스 기반 Projection (가장 가벼움)
public interface OrderSummary {
    Long getId();
    String getStatus();
}
List<OrderSummary> findByCustomerId(Long customerId);
```
> [[CQRS]]의 Query 측은 이렇게 **DTO/Projection을 직접 조회**해 엔티티 매핑 비용도 없앤다.

---

## 6. 매핑 전략 비교
| 방식 | 장점 | 단점 | 적합 |
|------|------|------|------|
| **수동(정적 팩토리)** | 명시적, 의존성 0, 디버깅 쉬움 | 보일러플레이트 | 소규모/단순 |
| **MapStruct** | 컴파일타임 생성, 빠름, 타입세이프 | 빌드 설정·애너테이션 | 중대형 ⭐ |
| **ModelMapper** | 런타임 자동 매핑 편함 | 리플렉션 느림·암묵적·오류 늦게 발견 | 비권장 추세 |
| **DTO 프로젝션** | 엔티티 안 거침, 조회 최적 | 쓰기엔 부적합 | 조회/[[CQRS]] |

---

## 7. 베스트 프랙티스
- **공개 API 경계에는 항상 DTO.** 엔티티 직접 노출 금지.
- 요청/응답 DTO를 **분리**(입력과 출력의 필드·검증이 다르다).
- DTO는 **불변(record)** 으로. 로직을 넣지 말 것(매핑 정적 메서드 정도는 허용).
- 검증(`@Valid`)은 **요청 DTO**에, 도메인 불변식은 엔티티에.
- 조회는 **프로젝션/Query DTO**로 over-fetching·N+1 회피.
- 대량 매핑은 **MapStruct**로 보일러플레이트 축소.
- 내부 계층(Service↔Domain)까지 무조건 DTO를 강요하지 말 것 — 경계가 아니면 도메인 객체로 충분.

## 8. 관련
- [[REST-API]] · [[gRPC]] · [[CQRS]] · [[JPA-Hibernate]] · [[DDD]] · [[QueryDSL]]
