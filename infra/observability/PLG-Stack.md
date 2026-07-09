---
tags:
  - observability
  - prometheus
  - loki
  - grafana
  - alloy
  - promtail
  - logging
  - metrics
created: 2026-06-16
---

# PLG 스택 — Prometheus + Loki + Grafana (+ Alloy)

> [!summary] 한 줄 요약
> **Grafana Labs의 오픈소스 관측성 스택**. Prometheus가 메트릭, Loki가 로그, Grafana가 통합 시각화를 담당한다. **Alloy**(구 Promtail/Agent)가 데이터 수집 에이전트. 경량·클라우드 네이티브·저비용이 강점.

---

## 1. 스택 구조

```
[애플리케이션/인프라]
  /actuator/prometheus  ← 메트릭 노출 엔드포인트
  stdout/파일 로그       ← 로그 출력

         │
  ┌──────▼───────┐
  │  Alloy Agent  │  ← 각 노드에 DaemonSet으로 배포
  │  (수집 에이전트) │    scrape 메트릭 + 로그 수집 + 전달
  └──┬────────┬──┘
     │        │
     ▼        ▼
[Prometheus]  [Loki]        ← 메트릭 저장 / 로그 저장 (인덱스 최소화)
     │            │
     └──────┬─────┘
            ▼
        [Grafana]           ← 단일 UI: 메트릭·로그 상관 분석
            │
        [Alertmanager]      ← 알림 라우팅 (Slack/PagerDuty)
```

---

## 2. Loki — 로그 저장소

### 핵심 설계 원칙

```
Loki는 로그를 인덱싱하지 않는다.
  → 라벨(Label)만 인덱싱 (job, namespace, pod, app)
  → 로그 내용은 압축된 청크로 저장
  → 검색 시: 라벨로 청크 선택 → LogQL로 full-scan

장점: 저장 비용 10~100배 절감 (vs Elasticsearch)
단점: 로그 내용 풀텍스트 검색 느림 (라벨로 범위 좁혀야 빠름)
```

### LogQL 기본

```logql
-- ── 로그 스트림 선택 ─────────────────────────────────────────────
{app="backend", namespace="production"}

-- ── 필터 (grep 역할) ─────────────────────────────────────────────
{app="backend"} |= "ERROR"             # 포함
{app="backend"} != "DEBUG"             # 제외
{app="backend"} |~ "timeout.*5432"     # 정규식

-- ── 파싱 (JSON 로그) ─────────────────────────────────────────────
{app="backend"} | json | level="ERROR"
{app="backend"} | json | duration > 1000   # 응답시간 1초 초과

-- ── 메트릭 변환 (로그 → 숫자) ────────────────────────────────────
# 분당 에러 로그 수
sum(rate({app="backend"} |= "ERROR" [5m])) by (namespace)

# P99 응답시간 (로그에서 추출)
quantile_over_time(0.99,
  {app="backend"} | json | unwrap duration [5m]
) by (app)
```

---

## 3. Alloy — 통합 수집 에이전트

Alloy는 2024년 Grafana Agent를 대체한 **범용 관측성 파이프라인**. Promtail(로그), Prometheus scrape, OpenTelemetry를 하나의 에이전트로 통합.

### Helm 설치

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install alloy grafana/alloy \
  --namespace monitoring --create-namespace \
  -f alloy-values.yaml
```

### Alloy 설정 (alloy-config.river)

```hcl
// ── 로그 수집 (K8s Pod 로그) ─────────────────────────────────────
loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.write.default.receiver]
}

discovery.kubernetes "pods" {
  role = "pod"
  namespaces { names = ["production", "staging"] }
}

// Pod 라벨 → Loki 라벨 매핑
loki.relabel "pods" {
  forward_to = [loki.write.default.receiver]
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "app"
  }
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
}

// ── 메트릭 scrape (Prometheus 호환) ─────────────────────────────
prometheus.scrape "spring_apps" {
  targets = [{
    __address__ = "backend:8080",
    app         = "backend",
  }]
  metrics_path   = "/actuator/prometheus"
  scrape_interval = "15s"
  forward_to     = [prometheus.remote_write.default.receiver]
}

// ── 전송 ─────────────────────────────────────────────────────────
loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}

prometheus.remote_write "default" {
  endpoint {
    url = "http://prometheus:9090/api/v1/write"
  }
}
```

---

## 4. Grafana — 통합 대시보드

### Explore에서 로그↔메트릭 상관 분석

```
Grafana Explore:
  1. Metrics 패널: rate(http_server_requests...)[5m] 에서 스파이크 발견
  2. "Logs for this time range" 클릭 → Loki로 자동 전환
  3. {app="backend"} |= "ERROR" 로 해당 시간대 에러 로그 확인
  → 단일 화면에서 원인 분석
```

### 주요 Grafana 플러그인 (PLG 연동)

| 플러그인 | 용도 |
|---------|------|
| **Tempo** | 분산 추적 (Jaeger 대체) |
| **Pyroscope** | 지속적 프로파일링 |
| **k6** | 부하 테스트 결과 시각화 |

---

## 5. 전체 Helm 배포 (Grafana LGTM 스택)

```bash
# kube-prometheus-stack: Prometheus + Grafana + Alertmanager
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=admin \
  --set prometheus.prometheusSpec.retention=15d

# Loki (단순 스케일러블 모드)
helm install loki grafana/loki \
  -n monitoring \
  --set loki.commonConfig.replication_factor=1 \
  --set loki.storage.type=filesystem

# Alloy
helm install alloy grafana/alloy \
  -n monitoring \
  --set controller.type=daemonset   # 모든 노드에 배포

# Grafana에 Loki 데이터소스 추가
# Settings → Data Sources → Add Loki → URL: http://loki:3100
```

---

## 6. 장단점

### ✅ 장점
- **저비용**: Loki는 인덱스를 최소화해 Elasticsearch 대비 스토리지 1/10
- **K8s 네이티브**: Helm 설치, DaemonSet 에이전트, K8s 라벨 자동 활용
- **단일 UI**: 메트릭·로그·추적을 Grafana 하나로 상관 분석
- **오픈소스 무료**: 클라우드 비용 없이 자체 호스팅
- **Prometheus 생태계**: 수천 개 exporter, PromQL 표준화

### ❌ 단점
- **로그 풀텍스트 검색 느림**: 인덱스 없어서 대용량 검색에 Elasticsearch보다 느림
- **복잡한 로그 분석 제한**: 집계·피벗·상관관계 분석은 ELK가 우세
- **Loki 쿼리 러닝커브**: LogQL 학습 필요, 라벨 설계 잘못하면 성능 저하
- **대규모 로그 처리**: Loki 클러스터 운영은 복잡 (분산 모드 필요)

---

## 7. 관련
- [[Prometheus-Grafana]] — Prometheus 상세 설정, PromQL, Alertmanager
- [[ELK-Stack]] — 풀텍스트 검색·복잡한 로그 분석이 필요할 때
- [[Metrics]] — Spring Boot 메트릭 노출
- [[Logging]] — 구조화 로그 (JSON)
- [[Tracing]] — 분산 추적
