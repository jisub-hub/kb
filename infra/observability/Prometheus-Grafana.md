---
tags:
  - observability
  - prometheus
  - grafana
  - alertmanager
  - monitoring
created: 2026-06-16
---

# Prometheus & Grafana — 메트릭 수집·시각화·알림

> [!summary] 한 줄 요약
> Prometheus가 앱과 인프라를 **pull 방식으로 scrape** → Grafana가 시각화 → Alertmanager가 알림 라우팅. Spring Boot는 `/actuator/prometheus` 엔드포인트로 메트릭을 노출한다.

---

## 1. 전체 스택 구조

```
[Spring Boot 앱]          [인프라]
  /actuator/prometheus     node_exporter (CPU/메모리/디스크)
  :8080                    kube-state-metrics (K8s 오브젝트 상태)
        │                        │
        └────────────────────────┘
                    │ scrape (pull, 15s 주기)
              [Prometheus]
                    │ query (PromQL)
                    ├─── [Grafana]  → 대시보드 / 시각화
                    │
                    └─── [Alertmanager]  → Slack / PagerDuty / Email
```

---

## 2. Helm으로 설치 (K8s)

```bash
# kube-prometheus-stack: Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics 올인원
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

---

## 3. prometheus.yml — scrape 설정

```yaml
# prometheus.yml (직접 설치 시)
global:
  scrape_interval:     15s    # 기본 scrape 주기
  evaluation_interval: 15s    # 알림 규칙 평가 주기
  scrape_timeout:      10s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - "alerts/*.yml"

scrape_configs:
  # Spring Boot 앱
  - job_name: "spring-backend"
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["backend:8080", "backend-2:8080"]
    # K8s에서는 annotation 기반 자동 디스커버리
    # kubernetes_sd_configs 사용

  # K8s 자동 디스커버리 (권장)
  - job_name: "k8s-pods"
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["production"]
    relabel_configs:
      # prometheus.io/scrape: "true" 어노테이션이 있는 파드만
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}

  # Node Exporter (시스템 메트릭)
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```

```yaml
# 파드 어노테이션 (자동 디스커버리 활성화)
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/actuator/prometheus"
    prometheus.io/port: "8080"
```

---

## 4. PromQL 핵심 쿼리

```promql
# ── 서비스 RED 메트릭 (Rate / Errors / Duration) ──────────────────

# 초당 요청 수 (5분 이동 평균)
rate(http_server_requests_seconds_count{application="order-service"}[5m])

# 5xx 에러율
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m])) * 100

# P99 응답시간 (히스토그램)
histogram_quantile(0.99,
  rate(http_server_requests_seconds_bucket{application="order-service"}[5m]))

# ── JVM ────────────────────────────────────────────────────────────

# Heap 사용률
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# GC 시간 비율
rate(jvm_gc_pause_seconds_sum[5m])  # 단위: 초/초

# 스레드 수
jvm_threads_live_threads

# ── 시스템 ────────────────────────────────────────────────────────

# CPU 사용률 (node exporter)
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)

# 메모리 사용 가능 비율
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# 디스크 사용률
1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
```

---

## 5. Alertmanager 설정

```yaml
# alerts/spring-boot.yml — 알림 규칙
groups:
  - name: spring-boot
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
        for: 2m          # 2분 이상 지속 시 발화
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.application }} 5xx 에러율 {{ $value | humanizePercentage }}"
          description: "5분간 에러율이 5%를 초과했습니다."

      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 응답시간 {{ $value }}초 초과"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.pod }} 크래시루프 발생"
```

```yaml
# alertmanager.yml — 알림 라우팅
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/..."

route:
  receiver: "slack-default"
  group_by: ["alertname", "application"]
  group_wait: 30s           # 같은 그룹 알림 모아서 발송
  group_interval: 5m
  repeat_interval: 1h       # 해결 안 되면 1시간마다 재알림
  routes:
    - match:
        severity: critical
      receiver: "pagerduty"
    - match:
        severity: warning
      receiver: "slack-default"

receivers:
  - name: "slack-default"
    slack_configs:
      - channel: "#alerts"
        title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        send_resolved: true

  - name: "pagerduty"
    pagerduty_configs:
      - routing_key: "..."
        description: '{{ .CommonAnnotations.summary }}'
```

---

## 6. Grafana 대시보드

```bash
# Grafana UI: localhost:3000 (admin/admin)
# 포트포워딩 (K8s)
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

### 권장 Import 대시보드 ID

| 대시보드 | ID | 설명 |
|---------|-----|------|
| **Spring Boot 2.1** | 11378 | HTTP, JVM, DB 커넥션 |
| **Spring Boot Statistics** | 6756 | 상세 JVM 메트릭 |
| **Node Exporter Full** | 1860 | 시스템 CPU/메모리/디스크/네트워크 |
| **Kubernetes Cluster** | 7249 | Pod, Namespace, Node 현황 |
| **JVM Micrometer** | 4701 | GC, 스레드, 메모리 상세 |

```
Grafana → Dashboards → Import → ID 입력 → Prometheus 데이터소스 선택
```

---

## 7. 관련
- [[Metrics]] — Spring 메트릭 설정
- [[Logging]] · [[Tracing]]
- [[../k8s/Kubernetes]] · [[../linux/Sysstat]]
