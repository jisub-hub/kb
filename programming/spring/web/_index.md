---
tags:
  - spring
  - web
  - moc
  - index
created: 2026-06-15
---

# 🌐 Web & 통신 MOC

> Spring 웹 스택과 서비스 간 통신.

## 웹 스택 모델
- [[MVC-vs-WebFlux]] — 블로킹(MVC) vs 논블로킹(WebFlux) 상세 비교 ⭐
- [[VirtualThreads-vs-WebFlux]] — MVC+가상스레드 vs WebFlux 성능 비교 ⭐

## API / 통신
- [[REST-API]] — HTTP 기반 REST 설계 & Spring 구현
- [[HTTP-Client-Resilience]] — RestClient/WebClient 회복성: 타임아웃 3종·클라이언트 풀·멱등 재시도·서킷 연동 ⭐
- [[gRPC]] — Protobuf/HTTP2 기반 고성능 RPC
- [[GraphQL]] — 클라이언트 주도 쿼리, BFF 패턴, DataLoader, Subscription
- [[Spring-Cloud-Gateway]] — 리액티브 API Gateway (라우팅·필터·인증)
- [[DTO]] — 데이터 전송 객체: 왜·언제 쓰는가 & 엔티티 직접 노출의 tradeoff ⭐
- [[Error-Handling]] — 에러 처리 & Problem Details(RFC 7807): @ControllerAdvice, 예외 계층, 검증 에러

## 관련
- [[Resilience4j]] · [[Sync-vs-Async-DataAccess]] · [[R2DBC]] · [[MSA]]
