---
tags:
  - infra
  - observability
  - monitoring
  - moc
  - index
created: 2026-06-16
---

# Observability MOC

> 인프라·애플리케이션 관측가능성 스택. 메트릭·로그·추적의 3축을 Prometheus/Grafana/Loki로 구성.

## 파일 구성

### 메트릭
- [[Prometheus-Grafana]] — Helm 설치, scrape_config, PromQL 핵심 쿼리, Alertmanager 룰, 추천 대시보드 ID
- [[SLO-SLI]] — SLI/SLO/SLA 구분, 에러 버짓·번레이트 알림, SLO 기반 알림, 배포 동결 정책

### 로그
- [[PLG-Stack]] — Prometheus + Loki + Grafana + Alloy 스택, LogQL, 경량·저비용
- [[ELK-Stack]] — Elasticsearch + Logstash + Kibana + Filebeat, 풀텍스트 검색·SIEM
- [[Logging-Strategy]] — 로깅 중요성, 감사로그 표준, 단일 vs 다중화 서버 로그 전략

## 애플리케이션 레이어 (Spring)
- [[../../programming/spring/observability/Metrics]] — Micrometer 설정, 커스텀 메트릭, SLI/SLO
- [[../../programming/spring/observability/Logging]] — 구조화 로그, MDC, 로그 레벨
- [[../../programming/spring/observability/Tracing]] — 분산 추적, OpenTelemetry, Zipkin/Tempo

## 시스템 레이어
- [[../linux/Sysstat]] — sar/iostat/mpstat/pidstat 장애 분석

## 관련
- [[../k8s/Kubernetes]] · [[../k8s/HPA]]
