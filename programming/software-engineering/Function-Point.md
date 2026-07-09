---
tags:
  - software-engineering
  - estimation
  - function-point
  - cost-estimation
  - metrics
created: 2026-06-17
---

# 기능 점수 산정 (Function Point Analysis)

> [!summary] 한 줄 요약
> 소프트웨어 규모를 **코드 줄 수(LOC)가 아니라 "사용자에게 제공하는 기능"의 양**으로 측정하는 기법. Albrecht(IBM, 1979)가 제안. 언어·기술과 무관하게 명세 단계에서 규모를 추정할 수 있어 **공수·비용·생산성 산정**에 쓰인다. 한국 공공 SW사업 대가산정의 표준 방식이다.

---

## 1. 왜 LOC가 아니라 FP인가

```
LOC(Lines of Code)의 문제:
  - 언어마다 같은 기능을 만드는 줄 수가 다름 (Assembly vs Python)
  - 코드 작성 전엔 알 수 없음 (사후 지표)
  - "줄 수 많음 = 생산성 높음"의 왜곡 (장황한 코드 보상)

FP의 장점:
  - 명세(요구사항)만으로 산정 가능 → 사전 견적
  - 언어·기술 독립적 → 프로젝트 간 비교
  - "사용자 관점 기능량" 측정 → 비즈니스 가치와 정렬
```
[Albrecht, 1979]

---

## 2. 5대 기능 요소

FP는 시스템을 두 종류 5개 요소로 분해한다.

### 트랜잭션 기능 (처리)
| 요소 | 의미 | 예 |
|------|------|-----|
| **EI** (External Input) | 외부에서 데이터를 받아 처리 | 등록·수정·삭제 폼 |
| **EO** (External Output) | 가공된 데이터를 외부로 출력 | 계산된 리포트, 통계 |
| **EQ** (External Inquiry) | 가공 없이 조회·검색 | 단순 조회 화면 |

### 데이터 기능 (저장)
| 요소 | 의미 | 예 |
|------|------|-----|
| **ILF** (Internal Logical File) | 시스템 내부가 유지하는 논리 데이터 | 자체 DB 테이블군(회원 마스터) |
| **EIF** (External Interface File) | 외부 시스템이 유지, 참조만 | 타 시스템 연동 데이터 |

---

## 3. UFP 계산 (미조정 기능점수)

각 요소를 **복잡도(단순/보통/복잡)**로 평가하고 가중치를 곱해 합산한다.

```
UFP = Σ (요소 개수 × 복잡도 가중치)

Albrecht 원 가중치(보통 복잡도 기준):
  EI  ×4    EO  ×5    EQ  ×4    ILF ×10   EIF ×7
```

```
예시:
  EI 5개  × 4  = 20
  EO 4개  × 5  = 20
  EQ 6개  × 4  = 24
  ILF 3개 × 10 = 30
  EIF 2개 × 7  = 14
  ───────────────────
  UFP = 108
```

> 실제 IFPUG는 각 요소별로 단순/보통/복잡 3단계 가중치 표(예 EI 3/4/6)를 사용한다. 위는 보통 기준 단순화.

---

## 4. VAF & AFP (복잡도 조정)

전통 IFPUG 방식은 14개 **일반 시스템 특성(GSC)**으로 보정한다.

```
GSC 14개: 데이터통신, 분산처리, 성능, 부하, 트랜잭션율, 온라인입력,
         사용자효율, 온라인갱신, 복잡처리, 재사용성, 설치용이성,
         운영용이성, 다중사이트, 변경용이성  (각 0~5점)

TDI(총 영향도) = Σ GSC (0~70)
VAF(조정인자)  = 0.65 + 0.01 × TDI   (0.65 ~ 1.35)

AFP(조정 기능점수) = UFP × VAF
```

> 최신 경향: IFPUG 카운팅에서 VAF/GSC는 **선택·축소**되는 추세(NESMA 간이법 등). 한국 SW사업 대가산정 가이드도 보정 없는 간이법/정통법을 구분한다.

---

## 5. FP → 공수·비용 추정

```
공수(man-month) = FP / 생산성(FP per man-month)
              또는 FP × (hours per FP)

Albrecht 보고: 1980 프로젝트 평균 18.3 시간/FP → 1981 평균 9.4 시간/FP (생산성 향상)

비용 = FP × FP당 단가
  한국 SW사업: "기능점수당 단가 × 보정계수"로 사업비 산정
  (소프트웨어 사업 대가산정 가이드, 공공 발주의 표준)
```

> FP는 규모(size) 지표다. 공수·비용은 여기에 생산성·난이도·팀 역량을 곱해야 나온다. COCOMO 등 추정 모델의 입력으로도 쓰인다.

---

## 6. 표준·방법론 계보

| 방식 | 특징 |
|------|------|
| **Albrecht (1979)** | 원조. 5요소 + 가중치 |
| **IFPUG** | 표준화 단체(1986~), 카운팅 매뉴얼(CPM), ISO/IEC 20926 |
| **NESMA** | IFPUG 호환 + 간이 추정법(early/quick) |
| **COSMIC** (ISO/IEC 19761) | 2세대. 데이터 이동(Entry/Exit/Read/Write) 기반, 실시간·임베디드에 적합 |
| **Mark II** | 영국, 트랜잭션 중심 |

---

## 7. 장단점·한계

```
장점: 기술 독립, 사전 산정, 프로젝트 간 비교, 발주·계약 근거
단점:
  - 카운팅에 전문성·표준 해석 필요 (주관 개입 여지)
  - UI/알고리즘 복잡도·비기능 요구를 잘 못 담음
  - 데이터 중심 업무시스템(SI)엔 적합, 알고리즘·게임·임베디드엔 부적합
    (그 경우 COSMIC 또는 다른 지표)
```

---

## 8. 실무 — 한국 공공 SI 맥락

```
- "소프트웨어 사업 대가산정 가이드"(한국SW산업협회)가 기능점수 방식을 규정
- 발주처는 FP로 사업 규모·예산 산정, 수주사는 FP로 견적·검수
- 정통법(상세 카운팅) vs 간이법(평균 복잡도 가정) 선택
- 감리·검수 시 FP 산정 내역이 산출물로 요구되기도 함
```

> 애자일 프로젝트에서는 FP 대신 **스토리 포인트(상대 추정)**를 쓰는 경우가 많다. FP는 절대적·계약적 산정, 스토리 포인트는 팀 내부 상대 추정 — 목적이 다르다. → [[Development-Methodologies]]

---

## 9. 참고문헌

[1] A. J. Albrecht, "Measuring Application Development Productivity," *Proc. IBM Applications Development Symposium*, Monterey, CA, pp. 83, Oct. 1979. — 기능점수의 기원.

[2] A. J. Albrecht, J. E. Gaffney, "Software Function, Source Lines of Code, and Development Effort Prediction: A Software Science Validation," *IEEE TSE*, vol. SE-9, no. 6, 1983.

[3] ISO/IEC 20926 — IFPUG Functional Size Measurement Method.

[4] ISO/IEC 19761 — COSMIC Functional Size Measurement.

[5] 한국소프트웨어산업협회, "소프트웨어 사업 대가산정 가이드".

---

## 관련
- [[Development-Methodologies]] — 산정과 방법론 (FP vs 스토리 포인트)
- [[Development-Deliverables]] — FP 산정 내역도 산출물
- [[UML-Modeling]] — 유스케이스/클래스로 기능 요소 식별
