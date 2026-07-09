---
tags:
  - database
  - mongodb
  - nosql
  - document-db
created: 2026-06-16
---

# MongoDB — 문서 지향 NoSQL

> [!summary] 한 줄 요약
> MongoDB는 **JSON 유사 BSON 문서를 컬렉션에 저장**하는 NoSQL DB. 스키마 유연성과 수평 확장(Sharding)이 강점. 관계형 데이터가 많거나 ACID 트랜잭션이 핵심이면 PostgreSQL이 낫다.

---

## 1. 핵심 개념

```
RDBMS         MongoDB
─────────     ─────────────
Database   ↔  Database
Table      ↔  Collection
Row        ↔  Document (BSON)
Column     ↔  Field
Primary Key↔  _id (ObjectId 자동 생성)
JOIN       ↔  $lookup (덜 권장) / 임베딩(권장)
Index      ↔  Index (동일)

[Document 예시]
{
  "_id": ObjectId("65a1..."),
  "userId": "user-123",
  "title": "주문 #1001",
  "items": [                         ← 배열 임베딩
    { "sku": "A01", "qty": 2, "price": 15000 },
    { "sku": "B03", "qty": 1, "price": 8000 }
  ],
  "address": {                       ← 서브 도큐먼트 임베딩
    "city": "서울", "zip": "04524"
  },
  "status": "SHIPPED",
  "createdAt": ISODate("2026-06-16T09:00:00Z")
}
```

---

## 2. 장단점

```
장점:
  ✅ 스키마 유연성 — 컬럼 변경 없이 필드 추가/제거
  ✅ 임베딩(denormalization) → 단일 쿼리로 관련 데이터 조회
  ✅ 수평 확장 (Sharding) 설계가 내장
  ✅ Aggregation Pipeline — 강력한 데이터 변환·집계
  ✅ Change Streams — 실시간 데이터 변경 감지
  ✅ Atlas Search — Elasticsearch 없이 전문 검색 가능
  ✅ 지리 정보 쿼리 내장 ($near, $geoWithin)
  ✅ JSON 그대로 저장 → REST API 백엔드와 자연스러운 통합

단점:
  ❌ JOIN 없음 → 복잡한 관계형 데이터 설계 어려움
  ❌ 트랜잭션: 4.0+부터 지원하지만 RDBMS보다 성능 저하
  ❌ 디스크 공간: BSON 오버헤드, 인덱스 크기
  ❌ 집계 쿼리 복잡도 (Aggregation Pipeline 러닝 커브)
  ❌ 운영 복잡도: Sharding 구성 어려움
```

---

## 3. 활용처

```
✅ 적합
  - 콘텐츠 관리 (블로그, 뉴스: 포스트마다 다른 필드)
  - 사용자 프로필 (다양한 SNS 연동 정보, 설정 다양)
  - 이벤트 로그 / 활동 기록 (스키마 변경 잦음)
  - IoT 센서 메타데이터 (디바이스마다 다른 필드)
  - 쇼핑카트 / 세션 저장
  - 카탈로그 데이터 (상품마다 속성 다름)
  - 실시간 게임 데이터

❌ 부적합
  - 강한 ACID 트랜잭션 필요 (금융 원장, 재고 차감)
  - 복잡한 JOIN이 필수인 정규화 데이터
  - 이미 있는 RDBMS로 충분한 경우 (무분별한 NoSQL 도입 금지)
```

---

## 4. 기본 CRUD

```javascript
// MongoDB Shell / Compass

// 삽입
db.orders.insertOne({
  userId: "user-123",
  items: [{ sku: "A01", qty: 2 }],
  status: "PENDING"
})

// 조회
db.orders.find({ status: "SHIPPED", "address.city": "서울" })
         .sort({ createdAt: -1 })
         .limit(20)

// 업데이트
db.orders.updateOne(
  { _id: ObjectId("...") },
  { $set: { status: "DELIVERED" }, $push: { log: "배달완료" } }
)

// 삭제
db.orders.deleteMany({ status: "CANCELLED", createdAt: { $lt: new Date("2025-01-01") } })
```

### Aggregation Pipeline

```javascript
// 월별 매출 집계
db.orders.aggregate([
  { $match: { status: "COMPLETED" } },
  { $unwind: "$items" },                                  // 배열 펼치기
  { $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$createdAt" } },
      totalRevenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
      orderCount: { $sum: 1 }
  }},
  { $sort: { _id: -1 } },
  { $limit: 12 }
])
```

---

## 5. 인덱스 전략

```javascript
// 단일 필드
db.orders.createIndex({ userId: 1 })

// 복합 인덱스 (쿼리 패턴 맞춤)
db.orders.createIndex({ userId: 1, createdAt: -1 })

// 텍스트 인덱스 (전문 검색)
db.products.createIndex({ name: "text", description: "text" })
db.products.find({ $text: { $search: "무선 키보드" } })

// TTL 인덱스 (자동 만료)
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })

// 희소 인덱스 (null 필드 제외)
db.users.createIndex({ phone: 1 }, { sparse: true })

// 인덱스 사용률 확인
db.orders.find({ userId: "u-1" }).explain("executionStats")
```

---

## 6. Spring Data MongoDB

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://user:pass@localhost:27017/mydb
```

```java
@Document(collection = "orders")
@Data
@Builder
public class Order {
    @Id
    private String id;               // ObjectId 자동 매핑

    private String userId;
    private List<OrderItem> items;   // 임베딩 (별도 컬렉션 X)
    private Address address;
    private OrderStatus status;

    @CreatedDate
    private Instant createdAt;
}

@Repository
public interface OrderRepository extends MongoRepository<Order, String> {
    List<Order> findByUserIdAndStatus(String userId, OrderStatus status);
    List<Order> findByCreatedAtBetween(Instant from, Instant to);

    // 커스텀 집계
    @Aggregation(pipeline = {
        "{ $match: { userId: ?0 } }",
        "{ $group: { _id: '$status', count: { $sum: 1 } } }"
    })
    List<StatusCount> countByStatus(String userId);
}
```

---

## 7. Replica Set (기본 HA 구성)

```yaml
# docker-compose.yml — Replica Set 3노드
services:
  mongo1:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports: ["27017:27017"]

  mongo2:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports: ["27018:27017"]

  mongo3:
    image: mongo:7
    command: mongod --replSet rs0 --bind_ip_all
    ports: ["27019:27017"]
```

```javascript
// mongo1에서 Replica Set 초기화
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})
```

---

## 8. 관련
- [[PostgreSQL-Internals]] — 관계형 DB 비교
- [[Elasticsearch]] — 전문 검색이 핵심이면 Elasticsearch
- [[TimescaleDB]] — 시계열 데이터는 TimescaleDB
- [[../observability/ELK-Stack]] — MongoDB 로그를 ELK로 수집
