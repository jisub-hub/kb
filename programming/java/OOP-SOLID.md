---
tags:
  - java
  - oop
  - solid
  - design-principles
created: 2026-06-15
---

# 객체지향 SOLID 원칙

> Robert C. Martin(Uncle Bob)이 정리한 5가지 객체지향 설계 원칙. 유지보수성·확장성·테스트 용이성의 기반.

---

## S — 단일 책임 원칙 (Single Responsibility Principle)

> **클래스는 변경 이유가 하나뿐이어야 한다.**

```java
// ❌ 위반: 주문 저장 + 이메일 발송을 한 클래스가 담당
class OrderService {
    void saveOrder(Order order) { /* DB 저장 */ }
    void sendConfirmationEmail(Order order) { /* 이메일 발송 */ }
}

// ✅ 분리
class OrderRepository { void save(Order order) { ... } }
class OrderNotifier  { void sendConfirmation(Order order) { ... } }
```

**실전 시그널**: 클래스를 설명할 때 "그리고"가 들어가면 SRP 위반 의심.

---

## O — 개방/폐쇄 원칙 (Open/Closed Principle)

> **확장에는 열려 있고, 수정에는 닫혀 있어야 한다.**

```java
// ❌ 위반: 새 결제 수단 추가 시 if-else 수정 필요
class PaymentProcessor {
    void pay(String type, int amount) {
        if (type.equals("CARD")) { ... }
        else if (type.equals("KAKAO")) { ... }
    }
}

// ✅ 전략 패턴으로 확장
interface PaymentStrategy { void pay(int amount); }
class CardPayment implements PaymentStrategy { ... }
class KakaoPayment implements PaymentStrategy { ... }

class PaymentProcessor {
    void pay(PaymentStrategy strategy, int amount) {
        strategy.pay(amount);  // 새 수단 추가 시 이 코드는 변경 없음
    }
}
```

---

## L — 리스코프 치환 원칙 (Liskov Substitution Principle)

> **자식 클래스는 부모 클래스를 완전히 대체할 수 있어야 한다.**

```java
// ❌ 위반: 정사각형이 직사각형을 상속하면 불변식이 깨짐
class Rectangle {
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    @Override void setWidth(int w)  { this.width = w; this.height = w; } // 부모 계약 위반
}

// ✅ 별도 추상화
interface Shape { int area(); }
class Rectangle implements Shape { ... }
class Square    implements Shape { ... }
```

**핵심**: 상속은 "is-a" 관계가 **행동 계약**까지 성립할 때만.

---

## I — 인터페이스 분리 원칙 (Interface Segregation Principle)

> **클라이언트는 사용하지 않는 메서드에 의존하지 않아야 한다.**

```java
// ❌ 위반: 프린터가 scan/fax 구현을 강요받음
interface Machine {
    void print();
    void scan();
    void fax();
}

// ✅ 역할별 분리
interface Printer { void print(); }
interface Scanner { void scan(); }
interface Fax     { void fax(); }

class SimplePrinter implements Printer { ... }
class AllInOne implements Printer, Scanner, Fax { ... }
```

---

## D — 의존성 역전 원칙 (Dependency Inversion Principle)

> **고수준 모듈은 저수준 모듈에 의존하지 말고, 둘 다 추상화에 의존해야 한다.**

```java
// ❌ 위반: 고수준(OrderService)이 저수준(MySQLOrderRepo) 구현에 직접 의존
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository();
}

// ✅ 추상화(인터페이스)에 의존 + 외부 주입 (DI)
interface OrderRepository { void save(Order order); }

class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo) { this.repo = repo; } // 생성자 주입
}

// Spring에서는 @Autowired / @RequiredArgsConstructor 로 해결
@Service
@RequiredArgsConstructor
class OrderService {
    private final OrderRepository repo; // 인터페이스 타입
}
```

---

## 원칙 간 관계 요약

| 원칙 | 핵심 질문 | 주로 해결하는 문제 |
|------|----------|-------------------|
| **SRP** | 이 클래스가 바뀌는 이유가 하나인가? | 거대 클래스, 뒤섞인 책임 |
| **OCP** | 기능 추가 시 기존 코드를 수정하는가? | if-else 분기 폭발 |
| **LSP** | 자식이 부모를 안전하게 대체하는가? | 잘못된 상속 계층 |
| **ISP** | 불필요한 메서드 의존이 없는가? | 비대한 인터페이스 |
| **DIP** | 구현이 아닌 추상화에 의존하는가? | 강결합, 테스트 불가 |

---

## Spring과 SOLID

- **SRP** → `@Service` / `@Repository` / `@Controller` 계층 분리
- **OCP** → `@Bean` 교체, 전략 패턴 + `@Qualifier`
- **LSP** → 인터페이스 기반 `@Repository` 구현체 교체 (JPA ↔ MongoDB)
- **ISP** → 도메인별 Repository 인터페이스 분리
- **DIP** → `@Autowired` 생성자 주입으로 구현 클래스 아닌 인터페이스 주입

## 관련
- [[Java-Patterns]] · [[Spring-Core]]
