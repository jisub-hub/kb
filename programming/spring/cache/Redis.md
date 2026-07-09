---
tags:
  - cache
  - redis
  - in-memory
  - nosql
created: 2026-06-15
---

# Redis

> [!summary] 한 줄 요약
> 인메모리 **key-value 데이터 스토어**. 캐시로 가장 널리 쓰이지만, 자료구조(List/Set/Hash/ZSet/Stream)·Pub/Sub·분산락·세션 저장 등 다용도. 단일 스레드 이벤트 루프 기반으로 매우 빠르다.

---

## 1. 사용처
| 용도 | 설명 |
|------|------|
| **캐시** | DB 앞단 조회 캐시 (가장 흔함) → [[Caching-Strategies]] |
| **세션 저장소** | 분산 환경 세션 공유 (Spring Session) |
| **Rate Limiting** | 토큰 버킷 카운터 ([[Spring-Cloud-Gateway]]) |
| **분산 락** | `SET NX PX` / Redlock |
| **랭킹/리더보드** | Sorted Set (ZSet) |
| **Pub/Sub, Stream** | 경량 메시징, 이벤트 ([[Kafka]]만큼 무겁지 않을 때) |
| **대기열/잡 큐** | List(LPUSH/BRPOP) |

## 2. 주요 자료구조
- **String**: 단순 값/카운터(`INCR`)
- **Hash**: 객체 필드 저장
- **List**: 큐/스택
- **Set / Sorted Set**: 중복제거 / 랭킹·범위 조회
- **Stream**: append-only 로그(소비자 그룹) — 경량 [[Kafka]] 대체 가능

---

## 3. Spring Data Redis 설정
```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-redis'   // Lettuce 기본
```
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      lettuce:
        pool:
          max-active: 16
          max-idle: 16
```

## 4. @Cacheable — 선언적 캐싱 (가장 쉬운 사용법)
```java
@EnableCaching                              // 메인 클래스 또는 설정에
@Configuration
public class CacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory cf) {
        var config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))                 // TTL
            .serializeValuesWith(SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()));
        return RedisCacheManager.builder(cf).cacheDefaults(config).build();
    }
}

@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository repository;

    @Cacheable(value = "product", key = "#id")          // 캐시에 있으면 메서드 미실행
    public Product getProduct(Long id) {
        return repository.findById(id).orElseThrow();
    }

    @CachePut(value = "product", key = "#product.id")   // 항상 실행 + 캐시 갱신
    public Product update(Product product) { return repository.save(product); }

    @CacheEvict(value = "product", key = "#id")         // 캐시 무효화
    public void delete(Long id) { repository.deleteById(id); }
}
```

## 5. RedisTemplate — 직접 제어
```java
@Service
@RequiredArgsConstructor
public class RankingService {
    private final StringRedisTemplate redis;

    public void addScore(String user, double score) {
        redis.opsForZSet().incrementScore("ranking", user, score);
    }
    public Set<String> top10() {
        return redis.opsForZSet().reverseRange("ranking", 0, 9);
    }
    // 분산 락
    public boolean tryLock(String key, Duration ttl) {
        return Boolean.TRUE.equals(
            redis.opsForValue().setIfAbsent(key, "1", ttl));   // SET NX PX
    }
}
```

## 6. 고가용성 / 확장
- **Replication**: master-replica 읽기 분산.
- **Sentinel**: 자동 failover.
- **Cluster**: 샤딩(16384 슬롯)으로 수평 확장.
- 매니지드: AWS ElastiCache, GCP Memorystore 등.

## 7. 운영 심화 (프로덕션) ⭐

> [!warning] Redis 실장애 3종: **eviction 정책 오선택**, **O(N) 명령 블로킹**, **cache stampede**
> Redis는 **싱글 스레드**(명령 처리)라, 한 개의 느린 명령이 전체를 멈춘다는 사실이 모든 함정의 뿌리다.

### 7.1 maxmemory-policy — 캐시냐 저장소냐로 갈린다

| 정책 | 동작 | 용도 |
|------|------|------|
| **noeviction** | 메모리 차면 쓰기 거부(에러) | 영속 저장소·큐(데이터 유실 불가) |
| **allkeys-lru / lfu** | 전체 키 중 LRU/LFU 제거 | **순수 캐시**(기본 선택) |
| **volatile-lru / ttl** | TTL 있는 키만 제거 | 캐시+영속 데이터 혼재 |

```
⚠️ 오선택 사고:
  캐시인데 noeviction → 메모리 차면 SET 전부 실패 → 장애
  저장소(세션·큐)인데 allkeys-lru → 멀쩡한 데이터가 조용히 증발
LFU(4.0+): 일회성 대량 조회가 핫 데이터를 밀어내는 LRU 약점 보완(접근 빈도 기준)
```

### 7.2 영속성 — AOF vs RDB

```
RDB (스냅샷): 주기적 덤프. 빠른 재시작·백업용. 마지막 스냅샷 이후 데이터 유실 가능
AOF (명령 로그): 모든 쓰기를 append. 내구성↑(fsync 정책), 파일 큼·재생 느림
권장: 둘 다 켜기(하이브리드, 4.0+) — AOF 베이스 + RDB 프리앰블로 빠른 로드
⚠️ fork 기반 저장(bgsave/AOF rewrite) → 메모리 2배 순간 사용·COW → 큰 인스턴스서 지연 스파이크
```

### 7.3 Cluster vs Sentinel — 다른 문제를 푼다

```
Sentinel: HA만 (자동 failover). 데이터는 단일 마스터에 전부 → 용량/처리량 한계
Cluster : 샤딩(16384 슬롯) = 수평 확장 + HA
  ⚠️ 멀티키 연산 제약: 여러 키가 다른 슬롯이면 MGET/MULTI/Lua 불가
     → 같은 슬롯에 묶으려면 hash tag: user:{123}:profile, user:{123}:cart ({} 안만 해싱)
선택: 용량/RPS가 단일 노드로 충분 → Sentinel, 넘으면 → Cluster
```

### 7.4 싱글스레드 = O(N) 명령이 전체를 블로킹 ⚠️

```
금지/주의 명령(프로덕션):
  KEYS *       → 전체 키 스캔, O(N) 블로킹 → 대신 SCAN(커서, 논블로킹)
  FLUSHALL/DB  → 동기 삭제 → ASYNC 옵션 또는 주의
  큰 컬렉션 DEL → 대신 UNLINK(백그라운드 회수)
  대형 SMEMBERS/HGETALL/LRANGE 0 -1 → 큰 응답이 이벤트루프 점유
진단: SLOWLOG GET, latency monitor, INFO commandstats
```

### 7.5 Big Key / Hot Key / Cache Stampede

```
Big Key : 수 MB 단일 키 → 조회/삭제가 블로킹 + 클러스터 슬롯 불균형
          → 분할(샤딩) 또는 구조 변경
Hot Key : 특정 키에 트래픽 집중 → 단일 샤드 과부하
          → 로컬 캐시(L1) 앞단·키 복제(replica 분산)
Cache Stampede(thundering herd): 인기 키 TTL 동시 만료 → DB로 요청 폭주
  대응: ① TTL 지터(만료 시각 분산)  ② 분산락으로 1개만 재생성([[Distributed-Lock]])
        ③ 논블로킹 사전 갱신(만료 전 백그라운드 리프레시)
Penetration: 없는 키 반복 조회 → null 캐싱·Bloom filter
Avalanche  : 대량 키 동시 만료/노드 다운 → TTL 분산·HA
```

## 8. 주의점
- **영속성 ≠ 주 저장소**: RDB/AOF로 영속화 가능하나, 기본은 캐시/휘발 가정(§7.2).
- **메모리 관리**: `maxmemory` + eviction 정책 — 용도에 맞게(§7.1).
- **Big Key / Hot Key** 주의 (단일 스레드라 큰 명령이 전체를 블로킹 → §7.4–7.5).
- **캐시 일관성**: 갱신 시 무효화 전략 명확히 → [[Caching-Strategies]].
- 라이선스: 7.4부터 RSALv2/SSPL → 완전 오픈소스 원하면 [[Valkey]].

## 9. 관련
- [[Valkey]] · [[Caching-Strategies]] · [[Distributed-Lock]] · [[Spring-Cloud-Gateway]] · [[CQRS]]
