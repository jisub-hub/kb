---
tags:
  - security
  - moc
  - index
created: 2026-06-15
---

# 🔐 Security MOC

> 인증(Authentication) / 인가(Authorization) / 토큰.

- [[Spring-Security]] — 인증·인가 프레임워크 (필터체인, 권한)
- [[OAuth2-JWT]] — OAuth2 / OIDC / JWT 토큰 기반 인증

## 관련
- [[Spring-Cloud-Gateway]] (게이트웨이 인증) · [[MSA]] · [[REST-API]] · [[gRPC]]

---

## 🧭 핵심 구분
| 용어 | 의미 |
|------|------|
| Authentication(인증) | "누구인가?" (로그인) |
| Authorization(인가) | "무엇을 할 수 있나?" (권한) |
| OAuth2 | 권한 위임 프로토콜(액세스 토큰 발급) |
| OIDC | OAuth2 위 인증 레이어(ID 토큰) |
| JWT | 자가포함(self-contained) 토큰 포맷 |
