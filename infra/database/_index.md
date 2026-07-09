---
tags:
  - infra
  - database
  - postgresql
  - moc
  - index
created: 2026-06-16
---

# Database MOC

> 데이터베이스 인프라 운영 — RDBMS, NoSQL, 클러스터링, 검색 엔진.

## 기초 & 개념
- [[DB-Fundamentals]] — ACID 원칙, CAP 이론, RDB vs NoSQL 유래와 비교
- [[DB-Deployment-Decision]] — 결정 트리: 관리형(RDS) vs VM+Patroni vs K8s Operator(CNPG) 비교

## 표준 & 모델링
- [[SSMS-용어사전]] — DB 물리 명명 표준(단어 19,623/용어 499/도메인 65), DDL 자동 교정(localhost:8800 + ddl-corrector 스킬)

## 관계형 DB (RDB)
- [[PostgreSQL-Internals]] — 인덱스 종류(B-Tree/GIN/BRIN), EXPLAIN ANALYZE 해석, VACUUM·autovacuum 튜닝, 파티셔닝
- [[pgvector]] — PostgreSQL 벡터 확장, HNSW/IVFFlat 인덱스, Spring AI 연동, 하이브리드 검색
- [[TimescaleDB]] — 시계열 하이퍼테이블, time_bucket 집계, 연속 집계, 압축, 보존 정책, Spring 연동
- [[PostgreSQL-HA-Cluster]] — Patroni + etcd + HAProxy 수동 HA 클러스터, 자동 Failover

## NoSQL
- [[MongoDB]] — 문서 지향 NoSQL, BSON, Aggregation Pipeline, Replica Set, Spring Data MongoDB
- [[Elasticsearch]] — 분산 검색 엔진, 역 인덱스, 한국어(nori), ILM, Spring Data ES

## 관련
- [[../../programming/spring/data/JPA-Hibernate]] · [[../../programming/spring/data/PgBouncer]]
- [[../../programming/spring/data/Connection-Pool]] · [[../../programming/spring/data/RDS-vs-Self-Hosted]]
- [[../../ai/RAG]] — pgvector를 RAG 파이프라인 벡터 DB로 활용
