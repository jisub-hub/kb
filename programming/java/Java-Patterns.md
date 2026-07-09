---
tags:
  - java
  - lambda
  - stream
  - reflection
  - functional
  - patterns
created: 2026-06-15
---

# Java 자주 쓰는 패턴 — Lambda · Stream · Reflection · Optional

> Java 8+ 에서 실전에서 자주 쓰이는 함수형·리플렉션 패턴 모음.

---

## 1. 함수형 인터페이스 — 언제, 왜, 어떻게 되는가

> 핵심 질문: **"이 코드 조각을 변수처럼 넘기고 싶은가?"** → 함수형 인터페이스 사용.

---

### Predicate\<T\> — `T → boolean` : 조건 판단을 분리할 때

**언제 쓰는가**
- "이 객체가 조건을 만족하는가" 판단 로직을 호출부에서 **주입**하고 싶을 때
- if문 조건이 반복되거나, 조건 자체를 변수/파라미터로 다뤄야 할 때
- `Stream.filter()` 에 넘길 때

**쓰면 어떻게 되는가**
- 조건 로직이 호출부에서 분리 → 조건 변경 시 메서드 내부를 건드리지 않아도 됨
- 여러 조건을 `.and()` / `.or()` / `.negate()` 로 조립 가능 → if-else 중첩 제거

```java
// ❌ 조건이 메서드에 박혀 있음 → 조건 바꾸려면 메서드 수정 필요
List<Order> findHighValueOrders(List<Order> orders) {
    return orders.stream()
        .filter(o -> o.getAmount() > 10000 && o.getStatus() == PAID)
        .toList();
}

// ✅ 조건을 파라미터로 분리 → 호출부에서 조건 교체 가능
List<Order> findOrders(List<Order> orders, Predicate<Order> condition) {
    return orders.stream().filter(condition).toList();
}

// 호출 예
Predicate<Order> highValue = o -> o.getAmount() > 10000;
Predicate<Order> paid      = o -> o.getStatus() == PAID;
Predicate<Order> vipOrder  = highValue.and(paid);

findOrders(orders, vipOrder);
findOrders(orders, highValue.negate());   // 소액 주문만
```

**실전 사용처**
- `Stream.filter(predicate)` — 컬렉션 필터링
- 입력값 유효성 검증 체인 (Bean Validation 보완)
- 리포지토리 동적 조건 조립 (`Specification` 패턴의 경량 대안)

---

### Function\<T, R\> — `T → R` : 변환 로직을 값으로 다룰 때

**언제 쓰는가**
- A 타입을 B 타입으로 변환하는 로직을 **교체 가능하게** 만들고 싶을 때
- `Stream.map()` 에 넘길 때
- 변환 단계를 조립(파이프라인)하고 싶을 때

**쓰면 어떻게 되는가**
- 변환 규칙이 코드가 아니라 **데이터**처럼 다뤄짐 → 런타임에 변환 방식 교체 가능
- `.andThen()` / `.compose()` 로 변환 파이프라인 구성

```java
// Entity → DTO 변환기를 파라미터로 받는 범용 매핑
<T, R> List<R> mapList(List<T> list, Function<T, R> mapper) {
    return list.stream().map(mapper).toList();
}

// 호출 예
List<OrderDto>     dtos    = mapList(orders, OrderDto::from);
List<String>       names   = mapList(orders, Order::getCustomerName);
List<BigDecimal>   amounts = mapList(orders, Order::getAmount);
```

```java
// 변환 파이프라인 — 단계별로 Function 조립
Function<String, String>  trim    = String::trim;
Function<String, String>  lower   = String::toLowerCase;
Function<String, String>  mask    = s -> s.replaceAll("(?<=.{3}).", "*");

Function<String, String> normalize = trim.andThen(lower).andThen(mask);

// "  Hong Gil Dong  " → "hon*********"
String result = normalize.apply("  Hong Gil Dong  ");
```

**실전 사용처**
- `Stream.map()` — 컬렉션 변환, Entity → DTO
- 전략 패턴에서 변환 전략 교체 (`Map<String, Function<...>>`)
- 파이프라인 처리 (입력 정규화, 응답 포맷 변환)

---

### Consumer\<T\> — `T → void` : 부작용만 있고 반환값이 없을 때

**언제 쓰는가**
- 값을 받아서 **저장·발행·출력** 하는 동작을 주입하고 싶을 때
- `Stream.forEach()` 에 넘길 때
- 콜백, 이벤트 핸들러 패턴

**쓰면 어떻게 되는가**
- 반환값이 없으므로 순수하게 **사이드이펙트** 전용
- `.andThen()` 으로 여러 소비 동작을 순서대로 실행 가능

```java
// 공통 처리 후 → 동작은 호출부가 결정
void processOrders(List<Order> orders, Consumer<Order> action) {
    orders.stream()
        .filter(o -> o.getStatus() == PENDING)
        .forEach(action);
}

// 호출 예 — 동작만 교체
processOrders(orders, orderRepository::save);           // DB 저장
processOrders(orders, eventPublisher::publish);         // 이벤트 발행
processOrders(orders, o -> log.info("처리: {}", o.getId())); // 로깅

// andThen — 저장하고 알림도 보내기
Consumer<Order> saveAndNotify =
    orderRepository::save
    .andThen(notificationService::send);
processOrders(orders, saveAndNotify);
```

**실전 사용처**
- `Stream.forEach()` — 컬렉션 순회 처리
- 콜백 등록 (`setOnSuccess(Consumer<T> callback)`)
- 이벤트 핸들러, 리스너 주입

---

### Supplier\<T\> — `() → T` : 값 생성을 나중으로 미룰 때 (지연 초기화)

**언제 쓰는가**
- 값이 **실제로 필요한 시점까지** 생성 비용을 미루고 싶을 때
- 기본값이 **비싼 연산**일 때 (`orElse`는 항상 실행, `orElseGet`은 비어 있을 때만)
- 팩토리 역할 — 호출마다 새 인스턴스가 필요할 때

**쓰면 어떻게 되는가**
- `orElse(value)` : Optional이 비어 있든 아니든 `value` **항상 평가됨**
- `orElseGet(supplier)` : Optional이 비어 있을 때만 supplier **호출됨** → 불필요한 연산 방지

```java
// ❌ orElse — Optional에 값이 있어도 DB 조회가 일어남
User user = optionalUser.orElse(userRepository.findDefault());
//                                 ↑ 항상 실행 → 불필요한 쿼리

// ✅ orElseGet — 없을 때만 호출
User user = optionalUser.orElseGet(() -> userRepository.findDefault());
//                                       ↑ Optional이 비어야 실행
```

```java
// 지연 초기화 패턴 — 처음 필요할 때 한 번만 생성
public class HeavyResourceHolder {
    private Supplier<HeavyResource> resourceSupplier =
        () -> HeavyResource.initialize();   // 실제 초기화는 get() 호출 시

    public HeavyResource get() {
        return resourceSupplier.get();
    }
}

// 팩토리 — 호출마다 새 객체
Supplier<UUID> newId = UUID::randomUUID;
Supplier<List<Order>> emptyCart = ArrayList::new;
```

**실전 사용처**
- `Optional.orElseGet(supplier)` — 비싼 기본값 지연 계산
- `Optional.orElseThrow(supplier)` — 예외 생성 지연
- 지연 초기화, 캐시 미스 시 로딩
- 테스트에서 픽스처 팩토리

---

### BiFunction\<T, U, R\> — `(T, U) → R` : 두 값을 조합할 때

**언제 쓰는가**
- 두 개의 다른 타입 입력을 받아 하나의 결과를 만들 때
- `Map.merge()`, `Map.compute()` 처럼 기존값 + 신규값 → 병합 결과

```java
// 두 소스를 합쳐 DTO 생성
BiFunction<Order, Member, OrderSummaryDto> buildSummary =
    (order, member) -> new OrderSummaryDto(
        order.getId(), member.getName(), order.getAmount()
    );

OrderSummaryDto dto = buildSummary.apply(order, member);

// Map.merge — 기존 값 있으면 합산, 없으면 신규 삽입
Map<String, Integer> wordCount = new HashMap<>();
words.forEach(w -> wordCount.merge(w, 1, Integer::sum));
//                                        ↑ BiFunction: (기존값, 1) → 합산
```

---

### UnaryOperator\<T\> — `T → T` : 같은 타입으로 변환할 때

> `Function<T,T>` 의 축약. 타입이 바뀌지 않는 변환에 의도를 명확히 표현.

```java
UnaryOperator<String> trim  = String::trim;
UnaryOperator<String> upper = String::toUpperCase;

// List.replaceAll — 리스트 원소를 일괄 변환
List<String> names = new ArrayList<>(List.of(" alice ", " bob "));
names.replaceAll(trim.andThen(upper));   // ["ALICE", "BOB"]

// 값 업데이트 파이프라인
UnaryOperator<BigDecimal> applyTax      = price -> price.multiply(BigDecimal.valueOf(1.1));
UnaryOperator<BigDecimal> applyDiscount = price -> price.subtract(BigDecimal.valueOf(500));
UnaryOperator<BigDecimal> finalPrice    = applyDiscount.andThen(applyTax);
```

---

### 인터페이스 선택 기준 요약

| 상황 | 선택 |
|------|------|
| "이 조건을 만족하는가?" → boolean 반환 | `Predicate<T>` |
| "A를 B로 변환" → 타입이 바뀜 | `Function<T,R>` |
| "A를 처리만 함" → 반환값 없음 | `Consumer<T>` |
| "나중에 꺼내쓸 값 생산" → 인자 없음 | `Supplier<T>` |
| "두 값 → 하나로 합침" | `BiFunction<T,U,R>` |
| "같은 타입 변환" → 타입 불변 | `UnaryOperator<T>` |

---

### 메서드 참조 — 람다를 더 짧게

```java
// 람다 → 메서드 참조 대응
order -> order.getId()       ⟺  Order::getId          // 인스턴스 메서드 (임의 객체)
s -> Integer.parseInt(s)     ⟺  Integer::parseInt      // 정적 메서드
() -> new ArrayList<>()      ⟺  ArrayList::new         // 생성자
e -> log.info(e)             ⟺  log::info              // 인스턴스 메서드 (특정 객체)
```

---

## 2. Stream API

### 파이프라인 구조
```
Source → intermediate ops (lazy) → terminal op (eager)
```

### 주요 중간 연산 (Intermediate)
```java
List<Order> orders = ...;

orders.stream()
    .filter(o -> o.getAmount() > 1000)          // Predicate
    .map(Order::getCustomerName)                 // Function
    .distinct()
    .sorted(Comparator.naturalOrder())
    .limit(10)
    .skip(2)
    .peek(System.out::println);                  // 디버깅용, 값 변경 금지
```

### flatMap — 중첩 컬렉션 평탄화
```java
List<List<String>> nested = List.of(List.of("a","b"), List.of("c","d"));
List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .toList();   // Java 16+
```

### 주요 종단 연산 (Terminal)
```java
// 수집
List<String>        list = stream.collect(Collectors.toList());
Set<String>         set  = stream.collect(Collectors.toSet());
Map<Long, Order>    map  = stream.collect(Collectors.toMap(Order::getId, o -> o));

// 그룹핑
Map<Status, List<Order>> byStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));

// 집계
long   count = stream.count();
int    sum   = intStream.sum();
OptionalInt max = intStream.max();

// 단락 연산
boolean any  = stream.anyMatch(pred);
boolean all  = stream.allMatch(pred);
boolean none = stream.noneMatch(pred);

Optional<T> first = stream.findFirst();
```

### reduce
```java
// 합산
int total = IntStream.rangeClosed(1, 100).reduce(0, Integer::sum);

// 커스텀 누산
Optional<String> joined = Stream.of("a","b","c")
    .reduce((a, b) -> a + "," + b);   // "a,b,c"
```

### Collectors 고급
```java
// 분류 + 카운트
Map<Status, Long> countByStatus = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()));

// joining
String csv = names.stream().collect(Collectors.joining(", ", "[", "]"));

// partitioningBy (true/false 분리)
Map<Boolean, List<Order>> partitioned = orders.stream()
    .collect(Collectors.partitioningBy(o -> o.getAmount() > 5000));
```

### 기본형 스트림 (박싱 비용 제거)
```java
IntStream.range(0, 10).sum();
LongStream.rangeClosed(1, 1_000_000).sum();
DoubleStream.of(1.1, 2.2, 3.3).average();

// 객체 스트림 ↔ 기본형 변환
OptionalDouble avg = orders.stream()
    .mapToInt(Order::getAmount)
    .average();
```

---

## 3. Optional

> null 반환 대신 **"값이 없을 수도 있음"**을 타입으로 표현.

```java
// 생성
Optional<String> present = Optional.of("value");
Optional<String> empty   = Optional.empty();
Optional<String> nullable = Optional.ofNullable(maybeNull);

// 꺼내기 (안전한 방법 우선)
String val = opt.orElse("default");
String val = opt.orElseGet(() -> computeDefault());
String val = opt.orElseThrow(() -> new NoSuchElementException());

// 조건부 소비
opt.ifPresent(System.out::println);
opt.ifPresentOrElse(v -> use(v), () -> log("empty"));   // Java 9+

// 변환
Optional<Integer> len = opt.map(String::length);
Optional<String>  sub = opt.flatMap(this::findSubValue);
Optional<String>  filtered = opt.filter(s -> s.length() > 3);

// 스트림 연계 (Java 9+)
opt.stream().forEach(System.out::println);
```

**주의**: `Optional`을 필드 타입, 메서드 파라미터, 컬렉션 원소로 쓰지 않는다. 반환 타입 전용.

---

## 4. Reflection

> 런타임에 클래스·메서드·필드 메타정보 탐색·조작. 프레임워크(Spring, JPA) 내부 핵심.

### 클래스 정보 획득
```java
Class<?> clazz = String.class;
Class<?> clazz = obj.getClass();
Class<?> clazz = Class.forName("java.lang.String");  // 동적 로딩
```

### 필드 탐색 및 접근
```java
Class<?> clazz = User.class;

// 선언된 모든 필드 (private 포함, 상속 제외)
Field[] fields = clazz.getDeclaredFields();
for (Field f : fields) {
    f.setAccessible(true);           // private 접근 허용
    Object value = f.get(userObj);
    System.out.println(f.getName() + " = " + value);
}
```

### 메서드 호출
```java
Method method = clazz.getDeclaredMethod("setName", String.class);
method.setAccessible(true);
method.invoke(userObj, "홍길동");
```

### 어노테이션 처리 (프레임워크 패턴)
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface NotNull {}

class User {
    @NotNull private String name;
}

// 런타임 검증
for (Field f : User.class.getDeclaredFields()) {
    if (f.isAnnotationPresent(NotNull.class)) {
        f.setAccessible(true);
        if (f.get(obj) == null)
            throw new ValidationException(f.getName() + " must not be null");
    }
}
```

### 동적 프록시 (JDK Dynamic Proxy)
```java
interface Service { String execute(String input); }

Service proxy = (Service) Proxy.newProxyInstance(
    Service.class.getClassLoader(),
    new Class[]{Service.class},
    (proxyObj, method, args) -> {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(realService, args);
        System.out.println("After: " + method.getName());
        return result;
    }
);
// → Spring AOP의 기반 원리
```

**Reflection 주의사항**
- 컴파일 타임 안전성 없음 → 런타임 예외 위험
- 성능 오버헤드 (캐싱 권장)
- `setAccessible(true)` 은 캡슐화 우회 → 꼭 필요한 경우만
- Java 17+ 모듈 시스템에서 `opens` 선언 없으면 접근 불가

---

## 5. 함수형 디자인 패턴

### Builder (with 람다)
```java
User user = User.builder()
    .name("홍길동")
    .email("hong@example.com")
    .age(30)
    .build();
// Lombok @Builder 로 자동 생성
```

### 전략 패턴 + 람다
```java
Map<String, Function<Order, BigDecimal>> discountStrategies = Map.of(
    "VIP",      order -> order.getTotal().multiply(BigDecimal.valueOf(0.9)),
    "COUPON",   order -> order.getTotal().subtract(BigDecimal.valueOf(1000)),
    "NORMAL",   order -> order.getTotal()
);

BigDecimal price = discountStrategies
    .getOrDefault(memberType, discountStrategies.get("NORMAL"))
    .apply(order);
```

### 체이닝 — Function compose / andThen
```java
Function<String, String> trim    = String::trim;
Function<String, String> toLower = String::toLowerCase;
Function<String, Integer> length = String::length;

Function<String, Integer> pipeline = trim.andThen(toLower).andThen(length);
int result = pipeline.apply("  Hello World  ");  // 11
```

---

## 패턴별 주요 사용처

| 패턴 | 실전 사용처 |
|------|------------|
| **Lambda** | 이벤트 핸들러, 전략 교체, 간단한 변환 |
| **Stream** | 컬렉션 필터링/변환/집계, DTO 매핑 |
| **Optional** | Repository 단건 조회 반환, null 연쇄 방지 |
| **Reflection** | 프레임워크 내부, 커스텀 어노테이션 처리, 직렬화 |
| **Dynamic Proxy** | AOP(로깅, 트랜잭션), 모킹(Mockito) |

## 관련
- [[OOP-SOLID]] · [[Java-09]] · [[Java-11-LTS]] · [[Java-16]]
