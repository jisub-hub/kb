---
tags:
  - collaboration
  - convention
  - code-style
  - naming
created: 2026-06-16
---

# 코딩 컨벤션 (Coding Convention)

> [!summary] 한 줄 요약
> 코드는 혼자 쓰지 않는다. 컨벤션은 **미래의 팀원(본인 포함)**을 위한 약속이다.

---

## 1. 왜 컨벤션이 필요한가

```
컨벤션 없는 팀:
  - PR 리뷰의 50%가 스타일 논쟁
  - 파일마다 다른 네이밍 → 탐색 시간 증가
  - 신규 입사자 온보딩 어려움

컨벤션 있는 팀:
  - 리뷰는 로직·설계에 집중
  - Lint/Formatter가 스타일 자동 강제
  - 어떤 파일을 열어도 익숙한 패턴
```

---

## 2. Java / Spring 네이밍 규칙

| 대상 | 규칙 | 예시 |
|------|------|------|
| 클래스/인터페이스 | PascalCase | `OrderService`, `UserRepository` |
| 메서드/변수 | camelCase | `createOrder()`, `userId` |
| 상수 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_PAGE_SIZE` |
| 패키지 | 소문자 + 점 구분 | `com.example.order.service` |
| 파일명 | 클래스명과 동일 | `OrderService.java` |
| DB 테이블/컬럼 | snake_case | `order_items`, `created_at` |
| REST API URL | kebab-case (복수 명사) | `/api/v1/order-items` |
| 환경변수 | UPPER_SNAKE_CASE | `DB_PASSWORD`, `JWT_SECRET` |

---

## 3. 메서드 네이밍 패턴

```java
// CRUD 동사 일관성
createUser()        // 생성
findUserById()      // 조회 (단건, 없으면 예외)
findUserByEmail()
findAllUsers()      // 조회 (목록)
findUsersByStatus() // 조건부 목록
updateUserEmail()   // 수정
deleteUser()        // 삭제

// Boolean 반환
isActive()          // is + 형용사
hasPermission()     // has + 명사
existsById()        // exists + 전치사

// 변환
toDto()             // 엔티티 → DTO
toEntity()          // DTO → 엔티티

// 이벤트/트리거
onOrderCreated()    // 이벤트 핸들러 (on + 과거분사)
```

---

## 4. 클래스 구조 순서

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    // 1. 상수
    private static final int MAX_ORDER_COUNT = 100;

    // 2. 의존성 (final 필드)
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;

    // 3. public 메서드 (외부 인터페이스)
    public OrderResponse createOrder(CreateOrderRequest request) { ... }
    public OrderResponse findOrderById(Long id) { ... }

    // 4. private 헬퍼 메서드
    private void validateOrderLimit(Long userId) { ... }
    private OrderItem buildOrderItem(CartItem cartItem) { ... }
}
```

---

## 5. 코드 스타일 규칙

### 들여쓰기 & 길이
```java
// ✅ 메서드 최대 20줄 목표
// ✅ 중첩 최대 2단계 (if 안의 if 안의 for → 추출)
// ✅ 줄 길이 최대 120자 (대부분의 IDE 기본)

// ❌ 나쁜 예: 3단계 이상 중첩
if (user != null) {
    if (user.isActive()) {
        for (Order order : orders) {
            if (order.isPending()) {
                // 복잡도 폭발
            }
        }
    }
}

// ✅ Early Return으로 평탄화
if (user == null) return;
if (!user.isActive()) return;
orders.stream()
      .filter(Order::isPending)
      .forEach(this::processOrder);
```

### 주석 작성 기준
```java
// ✅ 쓸 때: WHY가 코드로 자명하지 않을 때만
// PostgreSQL advisory lock — JPA 락이 Cross-JVM 환경에서 실패하는 이슈 우회
LockUtils.acquireAdvisoryLock(orderId);

// ❌ 쓰지 말 것: WHAT은 코드가 이미 말함
// 사용자 ID로 사용자를 찾는다
User user = userRepository.findById(userId);
```

---

## 6. Git Commit 메시지 (Conventional Commits)

```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

| type | 사용 시점 |
|------|-----------|
| `feat` | 새 기능 |
| `fix` | 버그 수정 |
| `docs` | 문서만 변경 |
| `refactor` | 동작 변경 없는 코드 개선 |
| `test` | 테스트 추가/수정 |
| `chore` | 빌드 설정, 의존성 업데이트 |
| `perf` | 성능 개선 |
| `style` | 포매팅, 세미콜론 등 (로직 변경 없음) |

```bash
# 예시
feat(order): 주문 취소 API 추가
fix(auth): JWT 만료 시 401 대신 500 반환되던 버그 수정
refactor(user): UserService 조회 메서드 중복 제거
docs(readme): 로컬 실행 가이드 추가
test(order): 동시 주문 초과 테스트 케이스 추가
```

---

## 7. PR 템플릿

```markdown
## 변경 사항
- [ ] 어떤 기능/수정인지 1~3줄 요약

## 관련 이슈
- closes #123

## 테스트
- [ ] 단위 테스트 통과
- [ ] 통합 테스트 통과
- [ ] 수동 테스트 시나리오: (직접 확인한 것 작성)

## 체크리스트
- [ ] 마이그레이션 스크립트가 있다면 포함
- [ ] API 변경 시 Swagger 문서 갱신
- [ ] 환경변수 추가 시 README·.env.example 반영
```

---

## 8. 자동화 도구

```yaml
# .editorconfig — IDE 공통 설정
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 4
end_of_line = lf
trim_trailing_whitespace = true
insert_final_newline = true

[*.yml]
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

```xml
<!-- Checkstyle: 빌드 시 스타일 강제 -->
<!-- Google Java Style Guide 기반 -->
<module name="Checker">
  <module name="LineLength">
    <property name="max" value="120"/>
  </module>
</module>
```

---

## 9. 관련
- [[Git-Workflow]] — 브랜치 전략, PR 프로세스
- [[Code-Review]] — 리뷰 기준, 피드백 문화
- [[Team-Communication]] — 보고 체계, 협업 문화
