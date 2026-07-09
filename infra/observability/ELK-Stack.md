---
tags:
  - observability
  - elasticsearch
  - logstash
  - kibana
  - filebeat
  - elk
  - logging
created: 2026-06-16
---

# ELK 스택 — Elasticsearch + Logstash + Kibana (+ Beats)

> [!summary] 한 줄 요약
> **Elastic사의 엔터프라이즈 로그 분석 플랫폼**. Elasticsearch가 풀텍스트 인덱싱·검색, Logstash/Filebeat가 수집·변환, Kibana가 시각화·분석 대시보드. 복잡한 로그 분석, 보안 감사, SIEM에 강점.

---

## 1. 스택 구조

```
[로그 소스]
  애플리케이션 로그 (stdout / 파일)
  시스템 로그 (/var/log/*)
  보안 이벤트 (audit, auth)
  네트워크 장비 syslog

         │
  ┌──────▼───────┐         ┌────────────────┐
  │   Filebeat    │ ──────► │   Logstash     │ ← 변환·필터링·강화
  │  (경량 에이전트) │         │ (파이프라인 처리) │   (선택 — 간단하면 생략)
  └──────────────┘         └───────┬────────┘
                                   │
                           ┌───────▼────────┐
                           │ Elasticsearch   │ ← 저장·인덱싱·검색
                           │  (클러스터)     │   샤드/레플리카 분산
                           └───────┬────────┘
                                   │
                           ┌───────▼────────┐
                           │    Kibana       │ ← 시각화·대시보드·알림
                           │  (UI + SIEM)   │   Lens, Canvas, ML
                           └────────────────┘
```

---

## 2. Elasticsearch 핵심 개념

```
Index   = 논리적 데이터 컨테이너 (RDB의 테이블)
Shard   = 인덱스의 물리 파티션 (기본 5개) → 수평 확장 단위
Replica = 샤드 복사본 → 고가용성 + 읽기 성능
Document = JSON 레코드 (RDB의 행)
Mapping  = 필드 타입 정의 (text/keyword/date/long...)
```

```
[1개 인덱스, Primary 2 + Replica 1]

노드 1: shard-0-primary  shard-1-replica
노드 2: shard-1-primary  shard-0-replica
```

### 인덱스 패턴 (시계열 로그)

```
logs-backend-2026.06.16   ← 일별 인덱스 (ILM으로 자동 롤오버)
logs-backend-2026.06.15
logs-backend-2026.06.14
       │
  ILM Policy: hot(7일) → warm(압축, 30일) → cold(90일) → delete
```

---

## 3. Filebeat 설정

```yaml
# filebeat.yml — K8s Pod 로그 수집
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true       # 어노테이션 기반 자동 설정
      hints.default_config:
        type: container
        paths:
          - /var/log/containers/*${data.kubernetes.container.id}.log

processors:
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
        in_cluster: true
  - decode_json_fields:
      fields: ["message"]
      target: ""                # 루트 레벨로 파싱
      overwrite_keys: true
  - drop_fields:
      fields: ["agent", "ecs"]  # 불필요 필드 제거

output.logstash:
  hosts: ["logstash:5044"]
# 또는 직접 Elasticsearch로:
# output.elasticsearch:
#   hosts: ["elasticsearch:9200"]
#   index: "logs-%{[kubernetes.namespace]}-%{+yyyy.MM.dd}"
```

---

## 4. Logstash 파이프라인

```ruby
# /etc/logstash/conf.d/spring-logs.conf

input {
  beats {
    port => 5044
  }
}

filter {
  # JSON 로그 파싱 (Spring Boot + Logback JSON appender)
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # 타임스탬프 파싱
  date {
    match => ["timestamp", "ISO8601", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }

  # 응답시간 숫자 변환
  if [duration_ms] {
    mutate {
      convert => { "duration_ms" => "integer" }
    }
  }

  # 민감 정보 마스킹
  mutate {
    gsub => [
      "message", "\b\d{6}-[1-8]\d{6}\b", "[SSN]",
      "message", "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}\b", "[EMAIL]"
    ]
  }

  # 슬로우 쿼리 태깅
  if [duration_ms] and [duration_ms] > 1000 {
    mutate {
      add_tag => ["slow_query"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}"
    # ILM 활성화
    ilm_enabled => true
    ilm_rollover_alias => "logs"
    ilm_pattern => "{now/d}-000001"
    ilm_policy => "logs-policy"
  }
}
```

---

## 5. Elasticsearch 인덱스 수명주기 관리 (ILM)

```json
// PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "7d"         // 7일 또는 50GB 초과 시 롤오버
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },   // 샤드 수 축소
          "forcemerge": { "max_num_segments": 1 }, // 세그먼트 병합 (압축)
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},              // 읽기 전용, 메모리 해제
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 6. Kibana — 주요 기능

### KQL (Kibana Query Language)

```kql
-- 에러 로그 검색
level: "ERROR" and kubernetes.namespace: "production"

-- 응답시간 1초 초과
duration_ms > 1000 and http.method: "POST"

-- 특정 시간대 특정 서비스
@timestamp >= "2026-06-16T09:00:00" and app: "order-service"

-- 와일드카드
message: "Connection refused*"

-- 필드 존재 확인
_exists_: exception.stacktrace
```

### Kibana Lens — 집계 시각화

```
1. Discover: 원시 로그 탐색, 타임라인 히스토그램
2. Lens: 드래그앤드롭 차트 생성
   - 에러 수 시계열 (date histogram + count)
   - 서비스별 평균 응답시간 (terms + avg)
   - Top N 느린 API (terms + 99th percentile)
3. Canvas: 커스텀 대시보드 (경영진 보고용)
4. Maps: IP 지오로케이션 시각화
```

### SIEM (보안 이벤트 관리)

```
Elastic Security:
  - 룰 기반 탐지 (Sigma 룰 임포트)
  - ML 기반 이상 탐지 (anomaly detection)
  - 타임라인 수사 도구
  - MITRE ATT&CK 매핑
```

---

## 7. Spring Boot 로그 JSON 포맷

```xml
<!-- logback-spring.xml — JSON 구조화 로그 (Logstash Encoder) -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>8.0</version>
</dependency>
```

```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <providers>
        <timestamp/>
        <logLevel/>
        <loggerName/>
        <message/>
        <stackTrace/>
        <mdc/>                          <!-- MDC 필드 자동 포함 -->
        <arguments/>
      </providers>
      <customFields>{"app":"order-service","env":"${SPRING_PROFILES_ACTIVE}"}</customFields>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="JSON"/>
  </root>
</configuration>
```

```java
// MDC로 요청 추적 컨텍스트 추가 (Servlet Filter)
@Component
public class MdcFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
            HttpServletResponse res, FilterChain chain) throws ... {
        try {
            MDC.put("traceId",  req.getHeader("X-Trace-Id"));
            MDC.put("userId",   SecurityContextHolder.getContext()...);
            MDC.put("clientIp", req.getRemoteAddr());
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}
```

---

## 8. Helm 설치 (K8s)

```bash
helm repo add elastic https://helm.elastic.co

# Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  -n logging --create-namespace \
  --set replicas=3 \
  --set volumeClaimTemplate.resources.requests.storage=100Gi \
  --set esJavaOpts="-Xmx4g -Xms4g"

# Kibana
helm install kibana elastic/kibana -n logging \
  --set elasticsearchHosts="http://elasticsearch-master:9200"

# Logstash
helm install logstash elastic/logstash -n logging \
  --set persistence.enabled=true

# Filebeat (DaemonSet)
helm install filebeat elastic/filebeat -n logging
```

---

## 9. 장단점

### ✅ 장점
- **강력한 풀텍스트 검색**: 역인덱스로 수십억 건에서도 빠른 검색
- **풍부한 집계**: Bucket/Metric/Pipeline 집계로 복잡한 분석
- **SIEM 기능**: 보안 이벤트 분석, 룰 엔진, ML 이상 탐지
- **성숙한 생태계**: 수백 개 Beats 에이전트, APM, RUM
- **Kibana UI**: 직관적인 Discover, 대시보드, Canvas

### ❌ 단점
- **고비용**: 스토리지 많이 사용 (Loki 대비 5~10배)
- **운영 복잡**: 샤드 설계, 클러스터 튜닝, JVM 힙 관리
- **리소스 소비**: Elasticsearch JVM 최소 4GB 권장 (Loki보다 무거움)
- **라이선스**: 일부 고급 기능(ML, SIEM) Elastic 라이선스 필요 (BSL 전환 이슈)
- **K8s 오버헤드**: 소규모 환경에서 리소스 낭비

---

## 10. PLG vs ELK 선택 기준

| 기준 | PLG (Loki) 선택 | ELK 선택 |
|------|----------------|---------|
| **인프라 비용** | 저비용 | 고비용 |
| **메트릭 + 로그 통합** | Grafana 하나로 | 별도 시스템 필요 |
| **로그 풀텍스트 검색** | 라벨 범위 내 검색 | 강력한 역인덱스 |
| **복잡한 로그 분석** | 제한적 | 강함 |
| **보안/감사/SIEM** | 기본 수준 | Elastic Security |
| **K8s 환경** | 최적화됨 | 사용 가능, 무거움 |
| **이미 사용 중인 스택** | Prometheus 있으면 | Elastic 라이선스 있으면 |
| **개발팀 규모** | 소~중규모 | 중~대규모 |

> 실무 패턴: **K8s + 클라우드 네이티브 → PLG**, **온프레미스 + 엔터프라이즈 보안 → ELK**

---

## 11. 관련
- [[PLG-Stack]] — PLG 스택 비교 및 Loki/Alloy 상세
- [[Prometheus-Grafana]] — Prometheus 메트릭 스택
- [[Logging]] — Spring 구조화 로그 설정
- [[../security/Cloud-Security-Products]] — SIEM 상용 제품 (Splunk, Sentinel)
