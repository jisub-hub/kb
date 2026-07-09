---
tags:
  - spring
  - graphql
  - api
  - query
created: 2026-06-16
---

# GraphQL

> [!summary] 한 줄 요약
> 클라이언트가 **필요한 데이터 구조를 직접 쿼리로 지정**하는 API 패러다임. REST의 over-fetching(불필요 데이터 수신)·under-fetching(여러 엔드포인트 호출) 문제를 해결하고, 단일 엔드포인트로 다양한 클라이언트 요구를 충족한다.

---

## 1. REST의 문제와 GraphQL의 해결

### Over-fetching
```
GET /api/users/1
→ { id, name, email, phone, address, createdAt, updatedAt, ... }
   // 화면엔 name만 필요한데 전부 내려옴
```

### Under-fetching (N+1 요청)
```
GET /api/posts/1          → { id, title, authorId }
GET /api/users/{authorId} → { name }   // 추가 호출 필요
```

### GraphQL 해결
```graphql
query {
  post(id: 1) {
    title
    author {       # 한 번의 요청으로 연관 데이터까지
      name
    }
  }
}
```

---

## 2. 핵심 개념

| 개념 | 설명 |
|---|---|
| **Schema** | 타입 시스템으로 API 형태를 정의하는 SDL(Schema Definition Language) |
| **Query** | 데이터 조회 (REST GET) |
| **Mutation** | 데이터 변경 (REST POST/PUT/DELETE) |
| **Subscription** | 실시간 이벤트 스트림 (WebSocket) |
| **Resolver** | 각 필드를 어떻게 가져올지 구현하는 함수 |
| **DataLoader** | N+1 문제를 배치 로딩으로 해결 |

---

## 3. 의존성 (Spring Boot 3.x)

```groovy
implementation 'org.springframework.boot:spring-boot-starter-graphql'
implementation 'org.springframework.boot:spring-boot-starter-web'  // MVC
// 또는
implementation 'org.springframework.boot:spring-boot-starter-webflux'  // WebFlux

// 개발용 GraphiQL UI (브라우저에서 쿼리 테스트)
// application.yml에서 활성화
```

```yaml
spring:
  graphql:
    graphiql:
      enabled: true        # /graphiql 에서 UI 접근 (개발 환경)
    path: /graphql         # 기본 엔드포인트
    schema:
      locations: classpath:graphql/   # .graphqls 파일 위치
      file-extensions: .graphqls, .gqls
```

---

## 4. 스키마 정의 (SDL)

`src/main/resources/graphql/schema.graphqls`

```graphql
type Query {
    post(id: ID!): Post
    posts(status: PostStatus, page: Int = 0, size: Int = 10): PostConnection!
    me: User
}

type Mutation {
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
}

type Subscription {
    commentAdded(postId: ID!): Comment!
}

type Post {
    id: ID!
    title: String!
    content: String!
    status: PostStatus!
    author: User!          # 연관 타입 → 별도 Resolver 필요
    comments: [Comment!]!
    createdAt: String!
}

type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
}

type Comment {
    id: ID!
    body: String!
    author: User!
    createdAt: String!
}

type PostConnection {
    content: [Post!]!
    totalElements: Int!
    totalPages: Int!
}

input CreatePostInput {
    title: String!
    content: String!
}

input UpdatePostInput {
    title: String
    content: String
    status: PostStatus
}

enum PostStatus {
    DRAFT
    PUBLISHED
    ARCHIVED
}
```

---

## 5. Controller — @QueryMapping / @MutationMapping

```java
@Controller
public class PostController {

    private final PostService postService;
    private final BatchLoaderRegistry loaderRegistry;

    // Query: post(id: ID!): Post
    @QueryMapping
    public Post post(@Argument Long id) {
        return postService.findById(id);
    }

    // Query: posts(status, page, size): PostConnection
    @QueryMapping
    public PostConnection posts(
            @Argument PostStatus status,
            @Argument int page,
            @Argument int size) {
        return postService.findAll(status, PageRequest.of(page, size));
    }

    // Mutation: createPost(input: CreatePostInput!): Post!
    @MutationMapping
    public Post createPost(@Argument CreatePostInput input) {
        return postService.create(input);
    }

    @MutationMapping
    public boolean deletePost(@Argument Long id) {
        postService.delete(id);
        return true;
    }

    // Post 타입의 author 필드 Resolver
    @SchemaMapping(typeName = "Post", field = "author")
    public User author(Post post) {
        return userService.findById(post.getAuthorId());
    }
    // ⚠️ 위 방식은 Post가 N개면 N번 호출 → N+1 문제 → DataLoader 사용 (섹션 7)
}
```

---

## 6. Subscription — 실시간 스트림

```java
@Controller
public class CommentController {

    private final Sinks.Many<Comment> commentSink = Sinks.many().multicast().onBackpressureBuffer();

    // Subscription: commentAdded(postId: ID!): Comment!
    @SubscriptionMapping
    public Flux<Comment> commentAdded(@Argument Long postId) {
        return commentSink.asFlux()
            .filter(c -> c.getPostId().equals(postId));
    }

    // 댓글 생성 시 구독자에게 발행
    @MutationMapping
    public Comment addComment(@Argument Long postId, @Argument String body) {
        Comment comment = commentService.create(postId, body);
        commentSink.tryEmitNext(comment);
        return comment;
    }
}
```

```yaml
# WebSocket 기반 Subscription 활성화
spring:
  graphql:
    websocket:
      path: /graphql-ws
```

---

## 7. DataLoader — N+1 문제 해결

Post가 100개면 `author` Resolver가 100번 호출되는 N+1 문제 발생.  
DataLoader는 개별 호출을 **배치로 모아** 한 번에 처리한다.

```java
@Configuration
public class DataLoaderConfig {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry(UserService userService) {
        BatchLoaderRegistry registry = new DefaultBatchLoaderRegistry();

        // "userLoader" 이름으로 배치 로더 등록
        registry.forTypePair(Long.class, User.class)
            .withName("userLoader")
            .registerBatchLoader((userIds, env) ->
                // userIds = [1, 2, 3, ...] 한 번에 모여 들어옴
                Flux.fromIterable(userService.findAllByIds(userIds))
            );
        return registry;
    }
}
```

```java
@SchemaMapping(typeName = "Post", field = "author")
public CompletableFuture<User> author(Post post, DataLoader<Long, User> userLoader) {
    // 즉시 DB 조회 X → 배치에 등록 후 한꺼번에 처리
    return userLoader.load(post.getAuthorId());
}
// 결과: N번 쿼리 → 1번 쿼리 (SELECT * FROM users WHERE id IN (...))
```

---

## 8. 에러 처리

```java
// GraphQL 에러는 HTTP 200으로 응답하면서 errors 배열에 담김
// 커스텀 예외 → GraphQL 에러 변환
@Component
public class GraphQLExceptionHandler implements DataFetcherExceptionResolver {

    @Override
    public Mono<List<GraphQLError>> resolveException(Throwable ex, DataFetchingEnvironment env) {
        if (ex instanceof PostNotFoundException e) {
            GraphQLError error = GraphqlErrorBuilder.newError(env)
                .message(e.getMessage())
                .errorType(ErrorType.NOT_FOUND)
                .build();
            return Mono.just(List.of(error));
        }
        return Mono.empty();  // 기본 처리로 위임
    }
}
```

**클라이언트가 받는 에러 응답:**
```json
{
  "data": { "post": null },
  "errors": [
    {
      "message": "Post not found: id=999",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["post"],
      "extensions": { "classification": "NOT_FOUND" }
    }
  ]
}
```

---

## 9. 인증·인가

```java
// Spring Security와 통합 — @PreAuthorize 그대로 사용
@Controller
public class PostController {

    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public Post createPost(@Argument CreatePostInput input) { ... }

    @MutationMapping
    @PreAuthorize("hasRole('ADMIN') or @postSecurity.isOwner(#id, authentication)")
    public boolean deletePost(@Argument Long id) { ... }
}
```

---

## 10. 쿼리 예시 (클라이언트)

```graphql
# 필요한 필드만 요청 (over-fetching 없음)
query GetPost {
  post(id: 1) {
    title
    author {
      name
    }
    comments {
      body
      author {
        name
      }
    }
  }
}

# 변수 사용
query GetPosts($status: PostStatus, $page: Int) {
  posts(status: $status, page: $page, size: 10) {
    content {
      id
      title
      createdAt
    }
    totalPages
    totalElements
  }
}

# 뮤테이션
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    id
    title
    createdAt
  }
}

# 구독
subscription OnComment($postId: ID!) {
  commentAdded(postId: $postId) {
    body
    author { name }
  }
}
```

---

## 11. REST vs gRPC vs GraphQL

| | REST | gRPC | GraphQL |
|---|---|---|---|
| 프로토콜 | HTTP/1.1 | HTTP/2 | HTTP/1.1 or 2 |
| 포맷 | JSON | Protobuf (바이너리) | JSON |
| 스키마 | OpenAPI (선택) | `.proto` (필수) | SDL (필수) |
| 클라이언트 제어 | ❌ 서버가 응답 형태 결정 | ❌ | ✅ 클라이언트가 필드 지정 |
| Over-fetching | 있음 | 있음 | ✅ 없음 |
| 실시간 | SSE / WebSocket 별도 | 스트리밍 내장 | Subscription 내장 |
| 브라우저 | ✅ 네이티브 | ❌ gRPC-Web 필요 | ✅ 네이티브 |
| 캐싱 | HTTP 캐시 (GET) | 어려움 | **복잡** (POST 단일 엔드포인트) |
| 디버깅 | curl 바로 가능 | protobuf 도구 필요 | GraphiQL UI |
| 학습 곡선 | 낮음 | 높음 | 중간 |
| 적합 | 공개 API, 일반 웹 | MSA 내부 고성능 | BFF, 복잡한 클라이언트 |

---

## 12. 언제 GraphQL을 선택하는가

### ✅ GraphQL이 유리한 상황
- **BFF(Backend For Frontend)** 패턴 — Web/iOS/Android가 다른 필드 조합이 필요할 때
- 클라이언트가 **다양한 데이터 조합**을 동적으로 요청하는 대시보드, 탐색 UI
- **모바일** 환경 — 네트워크 절약, 필요한 필드만 수신
- 프론트엔드 팀이 백엔드 의존 없이 **독립적으로 쿼리 설계** 원할 때

### ❌ REST/gRPC가 나을 때
- **단순한 CRUD** — GraphQL 오버헤드 불필요
- **파일 업로드/다운로드** — multipart는 GraphQL과 궁합이 나쁨
- **캐싱이 중요** — GraphQL은 POST 단일 엔드포인트라 HTTP 캐시 적용 어려움
- **MSA 내부 통신** — gRPC가 성능·타입 안전성 모두 우위
- **공개 외부 API** — REST가 범용성·문서화 면에서 유리

> [!tip] 실무 패턴: REST + GraphQL 혼용
> 외부 공개 API는 REST, 프론트엔드 전용 BFF 레이어만 GraphQL로 운영하는 경우 많음.

---

## 13. 관련
- [[REST-API]] · [[gRPC]] · [[Spring-Cloud-Gateway]] · [[MSA]]
