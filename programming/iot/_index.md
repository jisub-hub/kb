---
tags:
  - iot
  - moc
  - index
created: 2026-06-16
---

# IoT MOC

> IoT 디바이스에서 서버까지의 데이터 수집·처리·저장 파이프라인. LoRaWAN 저전력 광역망부터 MQTT 메시징, Node-RED 미들웨어, 시계열 저장까지.

## 데이터 수집 경로
- [[IoT-Data-Ingestion]] — Socket/TCP/UDP/MQTT/REST API/LoRaWAN 비교, 선택 기준
- [[MQTT]] — Pub/Sub 프로토콜 심화: QoS, LWT, Retained, MQTT 5.0, EMQX 설정
- [[LoRaWAN-ChirpStack]] — LoRa 물리층, LoRaWAN MAC, ChirpStack 설치·코덱·MQTT 연동

## 처리 미들웨어
- [[Node-RED]] — 비주얼 플로우 파이프라인, MQTT 수신·파싱·변환·라우팅·알림

## 전체 아키텍처
- [[IoT-Architecture]] — ChirpStack → Raw MQTT → Node-RED → 소프트웨어 브로커(EMQX) → 백엔드/DB

## 관련
- [[../../infra/database/TimescaleDB]] — 시계열 센서 데이터 저장
- [[../../infra/observability/PLG-Stack]] — 센서 메트릭 시각화 (Grafana)
- [[../spring/messaging/Kafka]] — 대용량 이벤트 파이프라인으로 확장 시
