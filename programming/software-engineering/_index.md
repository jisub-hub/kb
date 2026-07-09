---
tags:
  - software-engineering
  - sdlc
  - moc
  - index
created: 2026-06-17
---

# 🛠️ Software Engineering MOC

> 소프트웨어 공학 프로세스 — 어떻게 만들고(방법론), 얼마로 산정하고(FP), 어떻게 모델링하고(UML), 무엇을 남기나(산출물). 코딩 기술이 아닌 **개발 프로세스·관리** 영역.

## 방법론 & 산정
- [[Development-Methodologies]] — 폭포수(Royce 1970)·스파이럴(Boehm 1988)·애자일(Manifesto 2001)·Scrum/XP/Kanban, 비교·선택 기준 (학술 논문)
- [[Function-Point]] — 기능 점수 산정(Albrecht 1979): 5대 요소, UFP/VAF, 공수·비용 추정, IFPUG/COSMIC, 한국 SW 대가산정

## 모델링 & 산출물
- [[UML-Modeling]] — UML(OMG/Three Amigos): 구조(클래스/컴포넌트)·행위(유스케이스/시퀀스) 다이어그램, PlantUML/Mermaid
- [[Development-Deliverables]] — SDLC 단계별 산출물(SRS/설계서/테스트/매뉴얼), 방법론별 차이, 한국 SI 맥락

## 관련
- [[../spring/architecture/_index|Architecture & Patterns]] — 설계 단계 산출물의 실체
- [[../collaboration/_index|Collaboration]] — 팀 프로세스·보고 체계
- [[../spring/deploy/CICD]] · [[../spring/deploy/Deployment-Strategies]] — DevOps 실천

---

## 🧭 한눈에 — SE 프로세스 흐름
```
방법론 선택(폭포수/애자일) → 요구분석(UML 유스케이스, SRS)
  → 규모 산정(FP) → 설계(UML 클래스/시퀀스, 설계서)
  → 구현 → 테스트(테스트 산출물) → 배포 → 운영
전 과정의 결과물 = 산출물(Deliverables)
```
