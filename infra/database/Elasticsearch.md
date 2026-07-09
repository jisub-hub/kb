---
tags:
  - database
  - elasticsearch
  - search
  - nosql
  - full-text-search
created: 2026-06-16
---

# Elasticsearch — 분산 검색·분석 엔진

> [!summary] 한 줄 요약
> Elasticsearch는 **Apache Lucene 기반 분산 검색 엔진**. 전문 검색(Full-Text Search)·로그 분석·비정형 데이터 집계에 탁월. RDBMS나 MongoDB로 처리하기 어려운 **"contains", 랭킹 기반 검색, 복잡한 필터+집계 조합**이 필요할 때 사용.

---

## 1. 핵심 개념

```
RDBMS         Elasticsearch
─────────     ──────────────────
Database   ↔  Index (인덱스)
Table      ↔  (7.x+ 제거, 과거: Type)
Row        ↔  Document (JSON)
Column     ↔  Field
Primary Key↔  _id (자동 또는 지정)
Schema     ↔  Mapping (동적 생성 가능)
Index      ↔  Inverted Index (자동)
Partition  ↔  Shard

[클러스터 구조]
  Cluster
    ├── Node-1 (Master)
    │     ├── Primary Shard 0
    │     └── Replica Shard 1
    ├── Node-2 (Data)
    │     ├── Primary Shard 1
    │     └── Replica Shard 0
    └── Node-3 (Data)
          ├── Primary Shard 2
          └── Replica Shard 2
```

---

## 2. 장단점

```
장점:
  ✅ 역 인덱스(Inverted Index) → 수십억 문서에서 밀리초 전문 검색
  ✅ 관련도 점수(BM25) → 검색 결과 랭킹
  ✅ 한국어·형태소 분석 (nori 플러그인)
  ✅ 집계(Aggregation) — 히스토그램, 버킷, 메트릭 실시간 집계
  ✅ Near Real-Time (NRT) — 1초 내 색인 반영
  ✅ 수평 확장 — Shard 추가로 TB~PB 규모 처리
  ✅ RESTful API (HTTP + JSON, 언어 무관)
  ✅ Kibana와 통합 (ELK 스택)

단점:
  ❌ 저장 비용 높음 (역 인덱스 + 원본 _source 이중 저장)
  ❌ 쓰기 무거움 (세그먼트 병합, 색인 오버헤드)
  ❌ 트랜잭션 없음 (ACID 불가)
  ❌ JOIN 없음 → 관계형 데이터 비적합
  ❌ 운영 복잡도 (클러스터 설정, ILM, Shard 관리)
  ❌ 과도한 리소스 (기본 힙 최소 1GB)
  ❌ 실시간 업데이트 비효율 (삭제+재색인 방식)
```

---

## 3. 활용처

```
✅ 적합
  - 상품 검색 (이커머스 검색 핵심)
  - 로그 분석 (ELK 스택, APM)
  - 문서 검색 (법률 문서, 논문, 게시판)
  - 자동 완성 (suggest, completion)
  - 이상 탐지 (기계학습 X-Pack)
  - 한국어 형태소 분석 검색
  - 대시보드 집계 (매출 현황, 로그 통계)

❌ 부적합
  - 주문·결제 등 ACID 필요 데이터
  - 간단한 CRUD (과도한 선택)
  - 단순 키-값 조회 (Redis가 적합)
  - 정규화된 관계형 데이터
  - 비용이 중요한 소규모 서비스
    (단, "검색 기능"이 핵심이면 비용 감수)
```

---

## 4. 인덱스 설계 및 Mapping

```json
// 상품 검색 인덱스 매핑
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "korean": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["nori_part_of_speech", "lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "korean",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text", "analyzer": "korean" },
      "price":       { "type": "integer" },
      "category":    { "type": "keyword" },
      "tags":        { "type": "keyword" },
      "stock":       { "type": "integer" },
      "createdAt":   { "type": "date" }
    }
  }
}
```

---

## 5. 검색 쿼리

```json
// 한국어 전문 검색 + 필터 + 집계
POST /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "무선 키보드",
            "fields": ["name^3", "description"],
            "type": "best_fields",
            "fuzziness": "AUTO"
          }
        }
      ],
      "filter": [
        { "range": { "price": { "gte": 10000, "lte": 100000 } } },
        { "term": { "category": "전자기기" } },
        { "range": { "stock": { "gt": 0 } } }
      ]
    }
  },
  "sort": [
    { "_score": "desc" },
    { "price": "asc" }
  ],
  "from": 0,
  "size": 20,
  "aggs": {
    "price_range": {
      "histogram": { "field": "price", "interval": 10000 }
    },
    "by_category": {
      "terms": { "field": "category", "size": 10 }
    }
  },
  "highlight": {
    "fields": {
      "name": {},
      "description": { "fragment_size": 150 }
    }
  }
}
```

### 자동 완성 (Completion Suggest)

```json
// 매핑 추가
"name_suggest": {
  "type": "completion",
  "analyzer": "korean"
}

// 자동 완성 쿼리
POST /products/_search
{
  "suggest": {
    "product-suggest": {
      "prefix": "무선",
      "completion": {
        "field": "name_suggest",
        "size": 5
      }
    }
  }
}
```

---

## 6. Spring Data Elasticsearch

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: changeme
```

```java
@Document(indexName = "products")
@Setting(settingPath = "elasticsearch/settings.json")
@Data
@Builder
public class ProductDocument {
    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "korean")
    private String name;

    @Field(type = FieldType.Text, analyzer = "korean")
    private String description;

    @Field(type = FieldType.Integer)
    private Integer price;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Date, format = DateFormat.date_time)
    private Instant createdAt;
}

@Repository
public interface ProductSearchRepository
        extends ElasticsearchRepository<ProductDocument, String> {

    // 메서드명으로 쿼리 자동 생성
    List<ProductDocument> findByCategory(String category);
    List<ProductDocument> findByPriceBetween(int min, int max);
}

// 복잡한 쿼리는 ElasticsearchOperations 직접 사용
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations operations;

    public SearchHits<ProductDocument> search(String keyword, String category,
                                               int minPrice, int maxPrice) {
        var query = NativeQuery.builder()
            .withQuery(q -> q.bool(b -> b
                .must(m -> m.multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name^3", "description")))
                .filter(f -> f.term(t -> t
                    .field("category").value(category)))
                .filter(f -> f.range(r -> r
                    .field("price").gte(JsonData.of(minPrice))
                                   .lte(JsonData.of(maxPrice))))
            ))
            .withSort(Sort.by("_score").descending())
            .withPageable(PageRequest.of(0, 20))
            .withHighlightQuery(new HighlightQuery(
                new Highlight(List.of(new HighlightField("name"))), null))
            .build();

        return operations.search(query, ProductDocument.class);
    }
}
```

---

## 7. Spring에서 ES 사용하는 3가지 방식

Spring에서 Elasticsearch를 쓰는 방법은 JPA처럼 쓰는 **Spring Data ES**, 저수준 **Java Client**, 그리고 **Operator(ElasticsearchOperations)** 직접 호출로 나뉜다.

### 7-1. Spring Data ES — JPA처럼 쓰기 (방법1: 가장 단순)

```java
// @Document → @Entity, ElasticsearchRepository → JpaRepository 와 1:1 대응
@Document(indexName = "products")
public class ProductDocument { ... }

public interface ProductSearchRepository
        extends ElasticsearchRepository<ProductDocument, String> {

    // JPA 메서드명 쿼리와 동일 패턴
    List<ProductDocument> findByCategory(String category);
    List<ProductDocument> findByNameContaining(String keyword);
    Page<ProductDocument> findByPriceBetween(int min, int max, Pageable pageable);
}
```

```
장점: JPA 경험 그대로 적용. 빠른 개발.
단점: 복잡한 bool 쿼리, 집계, 하이라이트 → 메서드명으로 표현 불가.
적합: 단순 키워드 필터·범위 조회
```

---

### 7-2. ElasticsearchOperations — Operator 패턴 (방법2: 중간)

`ElasticsearchOperations`는 Spring Data의 **Template 패턴** 구현체. `JdbcTemplate` 과 유사한 포지션.

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations ops;   // Bean 자동 주입

    public SearchHits<ProductDocument> complexSearch(String keyword,
                                                      String category,
                                                      int minPrice, int maxPrice) {
        Query query = NativeQuery.builder()
            .withQuery(q -> q.bool(b -> b
                .must(m -> m.multiMatch(mm -> mm
                    .query(keyword)
                    .fields("name^3", "description")))
                .filter(f -> f.term(t -> t.field("category").value(category)))
                .filter(f -> f.range(r -> r
                    .field("price")
                    .gte(JsonData.of(minPrice))
                    .lte(JsonData.of(maxPrice))))
            ))
            .withAggregation("by_category", a -> a.terms(t -> t.field("category")))
            .withHighlightQuery(new HighlightQuery(
                new Highlight(List.of(new HighlightField("name"))), null))
            .withSort(Sort.by("_score").descending())
            .withPageable(PageRequest.of(0, 20))
            .build();

        return ops.search(query, ProductDocument.class);
    }

    // 인덱스 직접 관리
    public void createIndexWithMapping() {
        IndexOperations indexOps = ops.indexOps(ProductDocument.class);
        indexOps.createWithMapping();   // @Document, @Field 기반 자동 매핑
    }

    // 집계 결과 파싱
    public Map<String, Long> countByCategory(String keyword) {
        // ... NativeQuery with aggregation
        SearchHits<ProductDocument> hits = ops.search(query, ProductDocument.class);
        ElasticsearchAggregations aggregations = (ElasticsearchAggregations) hits.getAggregations();
        return aggregations.aggregationsAsMap()
            .get("by_category")
            .sterms()
            .buckets().array()
            .stream()
            .collect(toMap(b -> b.key().stringValue(), StringTermsBucket::docCount));
    }
}
```

```
장점: NativeQuery로 ES 전체 기능 사용 가능. Spring 생태계 통합.
단점: ES DSL과 Spring Wrapper 사이 번역 레이어 존재.
적합: 복잡한 검색, 집계, 하이라이트가 모두 필요한 경우
```

---

### 7-3. Java Client (Elasticsearch Java API Client) — 직접 호출 (방법3: 가장 저수준)

Spring Data 없이 공식 Java Client를 직접 사용. ES 8.x 기준 새로운 공식 클라이언트.

```xml
<dependency>
  <groupId>co.elastic.clients</groupId>
  <artifactId>elasticsearch-java</artifactId>
  <version>8.14.0</version>
</dependency>
```

```java
@Configuration
public class ElasticsearchClientConfig {

    @Bean
    public ElasticsearchClient elasticsearchClient() {
        RestClient restClient = RestClient.builder(
            new HttpHost("localhost", 9200)
        ).build();

        ElasticsearchTransport transport = new RestClientTransport(
            restClient, new JacksonJsonpMapper());

        return new ElasticsearchClient(transport);
    }
}

@Service
@RequiredArgsConstructor
public class RawEsService {

    private final ElasticsearchClient client;

    public SearchResponse<ProductDocument> rawSearch(String keyword) throws IOException {
        return client.search(s -> s
            .index("products")
            .query(q -> q
                .bool(b -> b
                    .must(m -> m.match(mt -> mt.field("name").query(keyword)))
                ))
            .size(20),
            ProductDocument.class
        );
    }

    // 직접 인덱스 생성
    public void createIndex() throws IOException {
        client.indices().create(c -> c
            .index("products")
            .settings(s -> s.numberOfShards("1").numberOfReplicas("1"))
            .mappings(m -> m
                .properties("name", p -> p.text(t -> t.analyzer("korean")))
                .properties("price", p -> p.integer(i -> i))
            )
        );
    }

    // 벌크 인덱싱
    public void bulkIndex(List<ProductDocument> products) throws IOException {
        List<BulkOperation> ops = products.stream()
            .map(p -> BulkOperation.of(b -> b
                .index(i -> i.index("products").id(p.getId()).document(p))
            ))
            .toList();

        BulkResponse response = client.bulk(b -> b.operations(ops));
        if (response.errors()) {
            // 실패 항목 처리
        }
    }
}
```

```
장점: ES API 100% 직접 접근. Spring 의존성 없음. 최신 ES 기능 즉시 사용 가능.
단점: Spring 통합 없음 (트랜잭션·AOP 미적용). 보일러플레이트 많음.
적합: Spring Data가 지원 안 하는 신규 ES 기능, 비Spring 환경, 성능 크리티컬한 직접 제어
```

---

### 방식 선택 기준

| 상황 | 권장 방식 |
|------|-----------|
| 단순 CRUD + 기본 검색 | Spring Data ES Repository |
| 복잡한 bool 쿼리 + 집계 | ElasticsearchOperations (NativeQuery) |
| ES 최신 기능 / Spring 없는 환경 | Java API Client |
| 혼합 (대부분 실무) | Repository로 기본, Operations로 복잡한 것 |

---

## 9. ILM (Index Lifecycle Management) — 로그 인덱스

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot":   { "actions": { "rollover": { "max_size": "50gb", "max_age": "7d" } } },
      "warm":  { "min_age": "7d",  "actions": { "shrink": { "number_of_shards": 1 } } },
      "cold":  { "min_age": "30d", "actions": { "freeze": {} } },
      "delete":{ "min_age": "90d", "actions": { "delete": {} } }
    }
  }
}
```

---

## 10. MongoDB vs Elasticsearch 선택

| 기준 | MongoDB | Elasticsearch |
|------|---------|--------------|
| 주 목적 | 문서 저장·CRUD | 검색·집계·분석 |
| 전문 검색 | 제한적 | ✅ 핵심 강점 |
| 쓰기 성능 | 빠름 | 느림 (역 인덱스 빌드) |
| 저장 비용 | 중간 | 높음 |
| 트랜잭션 | 4.0+ 지원 | ❌ 없음 |
| 한국어 검색 | Atlas Search | nori 플러그인 |

> 실무 패턴: **MongoDB가 원본 저장소, Elasticsearch는 검색 전용 뷰**. MongoDB 변경 → Change Stream → Elasticsearch 동기화.

---

## 11. 관련
- [[MongoDB]] — 원본 데이터 저장 + Elasticsearch 동기화 패턴
- [[../observability/ELK-Stack]] — 로그 파이프라인 (Filebeat → Logstash → ES)
- [[PostgreSQL-Internals]] — GIN 인덱스로 간단한 전문 검색 가능
