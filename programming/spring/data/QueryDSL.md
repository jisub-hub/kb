---
tags:
  - database
  - querydsl
  - jpa
  - sync
created: 2026-06-15
---

# QueryDSL

> [!summary] 한 줄 요약
> **타입 세이프(type-safe)** 한 **동적 쿼리** 빌더. 문자열 JPQL의 오타·런타임 오류 문제를 컴파일 타임에 잡아주고, 조건 조립을 코드로 깔끔하게 표현한다. 보통 [[JPA-Hibernate]]와 함께 사용.

---

## 1. 왜 쓰나
- 문자열 JPQL은 **오타가 런타임에야 발견**되고, 동적 조건(if 분기)을 붙이기 지저분하다.
- QueryDSL은 **Q타입(컴파일 생성 클래스)** 으로 필드를 참조 → IDE 자동완성 + 컴파일 검증.

## 2. 동적 쿼리 예시
```java
@RequiredArgsConstructor
public class OrderRepositoryImpl implements OrderRepositoryCustom {
    private final JPAQueryFactory queryFactory;

    public List<Order> search(OrderSearchCond cond) {
        QOrder order = QOrder.order;
        return queryFactory
            .selectFrom(order)
            .where(
                memberIdEq(cond.memberId()),   // null → 조건 자동 제외
                statusEq(cond.status()),
                createdAfter(cond.from()))
            .orderBy(order.createdAt.desc())
            .offset(cond.offset()).limit(cond.size())
            .fetch();
    }

    private BooleanExpression memberIdEq(Long id) {
        return id == null ? null : QOrder.order.member.id.eq(id);
    }
    private BooleanExpression statusEq(OrderStatus s) {
        return s == null ? null : QOrder.order.status.eq(s);
    }
    private BooleanExpression createdAfter(LocalDateTime t) {
        return t == null ? null : QOrder.order.createdAt.goe(t);
    }
}
```
> `where()`에 넘긴 `null` 조건은 자동으로 무시된다 → **동적 쿼리가 매우 깔끔**.

## 3. DTO 직접 프로젝션 ([[CQRS]] 읽기 모델에 유용)
```java
List<OrderSummaryDto> result = queryFactory
    .select(Projections.constructor(OrderSummaryDto.class,
            order.id, order.member.name, order.totalAmount))
    .from(order)
    .fetch();
```

## 4. 장점 / 단점
- ✅ 컴파일 타임 안전성, 동적 쿼리 간결, 자동완성/리팩터링 친화.
- ❌ Q타입 생성(빌드 설정) 필요, 의존성/빌드 복잡도 약간 증가.

## 5. 관련
- [[JPA-Hibernate]] · [[CQRS]] · [[MyBatis]]
