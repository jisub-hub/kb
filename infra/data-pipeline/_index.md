---
tags:
  - infra
  - data-pipeline
  - etl
  - moc
  - index
created: 2026-06-16
---

# Data Pipeline MOC

> 데이터 수집·변환·적재 파이프라인. 스케줄 기반 배치 처리와 워크플로우 오케스트레이션.

## 파일 구성
- [[Apache-Airflow]] — DAG 기반 ETL, Scheduler 주기 실행, TaskGroup 병렬화, BranchOperator

## 관련
- [[../../ai/N8N-Automation]] — n8n (비즈니스 자동화), LangChain, LangGraph
- [[../database/TimescaleDB]] — 시계열 데이터 적재 타겟
- [[../database/MongoDB]] — NoSQL 적재 타겟
