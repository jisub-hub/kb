---
tags:
  - spring
  - grpc
  - rpc
  - protobuf
  - api
created: 2026-06-15
---

# gRPC

> [!summary] 한 줄 요약
> Google이 만든 고성능 **RPC 프레임워크**. **Protocol Buffers(protobuf)** 로 서비스/메시지를 정의하고, **HTTP/2** 위에서 바이너리로 통신한다. 강타입 계약·양방향 스트리밍·낮은 오버헤드가 강점 → 내부 마이크로서비스 통신에 적합.

---

## 1. 특징
- **IDL 기반 계약 우선**: `.proto` 파일로 인터페이스 정의 → 다국어 코드 자동 생성.
- **HTTP/2**: 멀티플렉싱, 헤더 압축, **양방향 스트리밍**.
- **Protobuf 바이너리**: JSON보다 작고 빠름(직렬화/전송).
- **4가지 호출 방식**: Unary / Server-streaming / Client-streaming / Bidirectional.

## 2. 사용처
- **내부 서비스 간 통신**([[MSA]]) — 고성능·저지연.
- 실시간 스트리밍(텔레메트리, 채팅, 동기화).
- 폴리글랏 환경(다양한 언어 서비스 간 강타입 계약).
- ❌ 브라우저 직접 호출은 제약(gRPC-Web 필요), 공개 API엔 REST가 무난.

---

## 3. .proto 정의
```protobuf
syntax = "proto3";
option java_package = "com.example.order.grpc";
option java_multiple_files = true;

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);              // Unary
  rpc StreamOrders (StreamRequest) returns (stream OrderResponse);     // Server-stream
}

message GetOrderRequest { int64 id = 1; }
message OrderResponse {
  int64 id = 1;
  string customer_id = 2;
  string status = 3;
  int64 total_amount = 4;
}
```
> `.proto` 컴파일 시 서버 stub(추상 클래스) + 클라이언트 stub이 자동 생성된다.

## 4. Spring Boot 연동 (grpc-spring-boot-starter)
```groovy
implementation 'net.devh:grpc-spring-boot-starter:3.1.0.RELEASE'
implementation 'io.grpc:grpc-protobuf'
implementation 'io.grpc:grpc-stub'
// protobuf gradle 플러그인으로 .proto → 자바 코드 생성
```

### 서버 구현
```java
@GrpcService                                   // gRPC 엔드포인트 등록
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderService orderService;    // 기존 비즈니스 로직 재사용

    @Override
    public void getOrder(GetOrderRequest request,
                         StreamObserver<OrderResponse> responseObserver) {
        Order order = orderService.findById(request.getId());
        OrderResponse response = OrderResponse.newBuilder()
            .setId(order.getId())
            .setCustomerId(order.getCustomerId())
            .setStatus(order.getStatus().name())
            .setTotalAmount(order.totalAmount().amount())
            .build();
        responseObserver.onNext(response);       // 응답 전송
        responseObserver.onCompleted();          // 스트림 종료
    }
}
```
```yaml
grpc:
  server:
    port: 9090
```

### 클라이언트 호출
```java
@Service
public class OrderClient {
    @GrpcClient("order-service")                 // 채널 자동 주입
    private OrderServiceGrpc.OrderServiceBlockingStub stub;   // 동기 stub

    public OrderResponse getOrder(long id) {
        return stub.getOrder(GetOrderRequest.newBuilder().setId(id).build());
    }
}
```
```yaml
grpc:
  client:
    order-service:
      address: 'static://localhost:9090'        # 또는 discovery:///order-service
      negotiation-type: plaintext
```

### 서버 스트리밍 소비
```java
Iterator<OrderResponse> it = blockingStub.streamOrders(req);
while (it.hasNext()) { process(it.next()); }     // 스트림 순회
```

## 5. REST vs gRPC 비교

| 항목 | REST | gRPC |
|------|------|------|
| 프로토콜 | HTTP/1.1(주로) | HTTP/2 |
| 포맷 | JSON(텍스트) | Protobuf(바이너리) |
| 계약 | OpenAPI(선택) | `.proto`(필수, 강타입) |
| 스트리밍 | 제한적(SSE/WebSocket) | **양방향 기본 지원** |
| 성능 | 보통 | **높음**(작은 페이로드, 멀티플렉싱) |
| 브라우저 | 네이티브 | gRPC-Web 필요 |
| 가독성/디버깅 | 쉬움(curl) | 어려움(바이너리) |
| 적합 | 공개 API, 웹/모바일 | 내부 서비스 간, 고성능, 스트리밍 |

## 6. 베스트 프랙티스
- **proto 호환성**: 필드 번호 재사용 금지, 필드 삭제 대신 `reserved`.
- 외부엔 REST, 내부엔 gRPC (게이트웨이에서 변환) 하이브리드 흔함.
- 데드라인/타임아웃 설정 + [[Resilience4j]] 회복탄력성 병행.
- 에러는 gRPC status code(`NOT_FOUND`, `INVALID_ARGUMENT` 등) 사용.

## 7. 관련
- [[REST-API]] · [[MSA]] · [[Resilience4j]] · [[MVC-vs-WebFlux]]
