---
tags:
  - observability
  - moc
  - index
created: 2026-06-15
---

# 📊 Observability MOC

> 관측성 3대 축(Three Pillars): **Logs / Traces / Metrics**.

- [[Logging]] — 구조화 로깅, 중앙 집계
- [[Tracing]] — 분산 추적(요청 흐름)
- [[Metrics]] — 메트릭/모니터링/알림
- [[AOP-Logging]] — AOP 기반 Controller 감사 로그, MDC traceId, JWT Actor 추출, Slow Query 탐지

## 관련
- [[MSA]] · [[Resilience4j]] · [[Kubernetes]]

---

## 🧭 3대 축
| 축 | 질문 | 도구 |
|----|------|------|
| **Logs** | "무슨 일이 있었나?" | Loki, ELK, Fluentd |
| **Traces** | "요청이 어디서 느렸나?" | Tempo, Jaeger, Zipkin |
| **Metrics** | "시스템 상태 수치는?" | Prometheus, Grafana |

> OpenTelemetry(OTel)가 세 축을 아우르는 표준으로 자리잡는 중.
