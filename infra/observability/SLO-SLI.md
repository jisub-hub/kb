---
tags:
  - observability
  - slo
  - sli
  - sla
  - error-budget
  - sre
created: 2026-06-17
---

# SLO / SLI & 에러 버짓

> [!summary] 한 줄 요약
> **SLI**는 측정값(가용성·지연·에러율), **SLO**는 그 목표(99.9%), **SLA**는 어기면 배상하는 계약. SLO는 100%가 아니라 **의도적으로 실패 여유(에러 버짓)**를 남긴다. 에러 버짓이 남으면 빠르게 배포하고, 소진되면 배포를 멈추고 안정화한다. 임계값 알림보다 **번 레이트(burn rate) 알림**이 신호 대 소음비가 좋다.

---

## 1. SLI / SLO / SLA 구분

```
SLI (Service Level Indicator) — 지표
  서비스 품질을 나타내는 정량적 측정값
  형식: "좋은 이벤트 수 / 전체 유효 이벤트 수" (비율)
  예: 가용성 = 2xx·3xx·4xx 응답 / 전체 요청
      지연   = 200ms 내 응답 / 전체 응답

SLO (Service Level Objective) — 목표
  SLI에 대해 내부적으로 정한 목표치 (시간 창 포함)
  예: "30일 롤링 윈도우에서 가용성 99.9%"

SLA (Service Level Agreement) — 계약
  고객과 맺는 외부 계약. 위반 시 배상/크레딧 발생
  보통 SLO보다 느슨하게 설정 (SLA 99.5% < SLO 99.9%)
  → 내부 SLO를 먼저 위반하면서 경보를 받아, SLA 위반 전에 대응
```

| 구분 | 대상 | 어기면 | 누가 정하나 |
|------|------|--------|------------|
| **SLI** | 측정값 | (측정일 뿐) | 엔지니어링 |
| **SLO** | 내부 목표 | 에러 버짓 소진·배포 중단 | 엔지니어링 + 제품 |
| **SLA** | 외부 계약 | 환불·크레딧·신뢰 손상 | 영업/법무 + 제품 |

---

## 2. SLI 선택 — RED / USE

좋은 SLI는 **사용자 경험과 직결**되고, **비율(0~1)**로 정규화된다. 무엇을 측정할지는 RED(요청 기반 서비스)·USE(리소스 기반)로 고른다.

```
RED — 요청 처리 서비스 (API, 웹)
  Rate      초당 요청 수
  Errors    실패 요청 비율          → 가용성 SLI 후보
  Duration  응답 시간 분포(P50/P99) → 지연 SLI 후보

USE — 리소스 (CPU·메모리·디스크·큐)
  Utilization  사용률
  Saturation   포화도(대기 큐 길이)
  Errors       에러 수

[SLI 카테고리별 정의 예]
  가용성  = (전체 요청 − 5xx) / 전체 요청
  지연    = (300ms 내 응답 수) / 전체 응답 수
  품질/정합성 = 정상 처리 / 전체 (배치·파이프라인)
  신선도  = 신선한 데이터 응답 / 전체 (캐시·복제)
```

> [!tip] SLI는 "사용자가 느끼는 것"을 측정
> 서버 CPU 90%는 그 자체로 SLI가 아니다(사용자는 모름). CPU가 높아 응답이 느려지는 것 — 즉 **지연 SLI**가 사용자 신호다. USE 지표는 원인 분석용, SLO는 RED 기반 사용자 신호로 잡는다.

---

## 3. 에러 버짓 계산

```
에러 버짓 = (1 − SLO) × 시간 창

  허용 다운타임 (가용성 SLO, 30일 기준):
    99%     → 7.2시간 / 월   (1 − 0.99)
    99.9%   → 43.2분 / 월    ← "쓰리 나인"
    99.95%  → 21.6분 / 월
    99.99%  → 4.32분 / 월    ("포 나인")
    99.999% → 25.9초 / 월    ("파이브 나인")

[요청 기반 환산]
  월 1억 요청, SLO 99.9% (에러율 0.1%)
  → 에러 버짓 = 100,000,000 × 0.001 = 100,000 요청
  → 이 10만 건이 "실패해도 되는 예산"
```

핵심 사고방식: **SLO 100%는 목표가 아니다.** 100%를 목표하면 변경을 두려워하게 되고, 비용이 비현실적으로 커진다. 의도적으로 남긴 에러 버짓은 **배포·실험·위험 감수의 재원**이다.

---

## 4. 번 레이트(Burn Rate) 알림

번 레이트 = 에러 버짓을 **소진하는 속도의 배수**. 1이면 정확히 시간 창 끝에 버짓을 다 쓰는 속도, 14.4면 그 속도의 14.4배(아주 빠름).

```
                  실제 에러율
  burn rate = ─────────────────
              (1 − SLO) = 허용 에러율

  예: SLO 99.9% (허용 0.1%), 실제 1% 에러 → burn rate = 10
      한 달치 버짓을 약 3일 만에 소진하는 속도
```

| 알림 종류 | 윈도우 | 번 레이트 임계 | 의미 | 액션 |
|----------|--------|---------------|------|------|
| **빠른 소진** | 1h (+ 5m 보조) | ≥ 14.4 | 2일이면 한 달 버짓 소진 | 즉시 페이징 |
| **느린 소진** | 6h (+ 30m 보조) | ≥ 6 | 며칠 내 소진 | 티켓·근무 시간 대응 |

```
[다중 윈도우 / 다중 번레이트 — 구글 SRE 권장]
  긴 윈도우  : 지속적 문제인지 판단 (오탐 억제)
  짧은 윈도우: 이미 회복됐는지 판단 (빠른 해제)
  → 두 윈도우가 동시에 임계 초과할 때만 발화
     = 빠른 감지 + 낮은 오탐 + 빠른 리셋
```

---

## 5. 왜 SLO 기반 알림이 임계값 알림보다 나은가

```
[임계값 알림의 문제]
  "CPU > 80%" "에러 5분간 5% 초과" 같은 고정 임계
  ❌ 증상이 아닌 원인을 알림 → CPU 높아도 사용자 정상이면 노이즈
  ❌ 짧은 스파이크에 과민 발화 → 알림 피로 → 무시 → 진짜를 놓침
  ❌ 트래픽 규모를 모름 → 새벽 1건 실패와 피크 1만 건 실패가 동급

[SLO 번레이트 알림의 장점]
  ✅ 사용자 영향(버짓 소진)에 비례해 발화
  ✅ 소진 속도로 긴급도 구분 (빠름 = 페이징, 느림 = 티켓)
  ✅ 다중 윈도우로 오탐·플래핑 억제
  ✅ "얼마나 심각한가"를 시간 예산으로 직관 표현
```

---

## 6. Prometheus / Grafana 구현

SLI 비율을 매번 계산하면 무겁다. **recording rule**로 미리 집계하고, 번 레이트는 여러 윈도우의 비율로 계산한다.

```yaml
# rules/slo.yml — recording rules
groups:
  - name: slo-order-service
    interval: 30s
    rules:
      # 좋은 요청 / 전체 요청 (가용성 SLI)
      - record: job:slo_errors:ratio_rate5m
        expr: |
          sum(rate(http_server_requests_seconds_count{job="order-service",status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count{job="order-service"}[5m]))
      - record: job:slo_errors:ratio_rate1h
        expr: |
          sum(rate(http_server_requests_seconds_count{job="order-service",status=~"5.."}[1h]))
          / sum(rate(http_server_requests_seconds_count{job="order-service"}[1h]))
      - record: job:slo_errors:ratio_rate6h
        expr: |
          sum(rate(http_server_requests_seconds_count{job="order-service",status=~"5.."}[6h]))
          / sum(rate(http_server_requests_seconds_count{job="order-service"}[6h]))
```

```yaml
# alerts/slo.yml — 다중 윈도우 번레이트 알림 (SLO 99.9% → 허용 0.001)
groups:
  - name: slo-burn-rate
    rules:
      - alert: ErrorBudgetBurnFast      # 1h ≥ 14.4× AND 5m 보조 확인
        expr: |
          job:slo_errors:ratio_rate1h > (14.4 * 0.001)
          and job:slo_errors:ratio_rate5m > (14.4 * 0.001)
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "에러 버짓 급속 소진 (burn rate ≥ 14.4) — 즉시 대응"

      - alert: ErrorBudgetBurnSlow      # 6h ≥ 6×
        expr: |
          job:slo_errors:ratio_rate6h > (6 * 0.001)
        for: 15m
        labels: { severity: warning }
        annotations:
          summary: "에러 버짓 완만 소진 (burn rate ≥ 6) — 티켓 대응"
```

```promql
# Grafana — 남은 에러 버짓 비율 (30일 윈도우)
1 - (
  sum(increase(http_server_requests_seconds_count{job="order-service",status=~"5.."}[30d]))
  / sum(increase(http_server_requests_seconds_count{job="order-service"}[30d]))
) / 0.001
# 결과 1.0 = 버짓 100% 남음, 0 = 전부 소진
```

스택 설치·PromQL 기본은 [[Prometheus-Grafana]] 참고.

---

## 7. 에러 버짓 소진 시 정책

```
[에러 버짓 정책 — 사전 합의가 핵심]

  버짓 남음 (정상)
    → 자유롭게 기능 배포·실험·위험 감수
    → 신뢰성에 과투자하지 않음 (속도 우선)

  버짓 소진 (SLO 위반 임박/위반)
    → 기능 배포 동결(freeze) — 신뢰성 작업만 허용
    → 장애 원인 수정·테스트·복원력 강화 우선
    → 버짓이 회복될 때까지 유지

  반복 소진
    → SLO 재검토 (목표가 비현실적인가?) 또는
       아키텍처 개선(부하 분산·격리·재시도 백오프)
```

이 정책은 **개발(속도)과 운영(안정)의 갈등을 데이터로 중재**한다. "더 빨리 배포하자 vs 더 안정적으로"를 감정이 아니라 에러 버짓 잔량으로 결정한다. 부분 장애 격리는 [[Resilience4j]](서킷브레이커·레이트리미터), 원인 추적은 [[Logging-Strategy]]와 연계한다.

---

## 관련
- [[Prometheus-Grafana]] — 메트릭 수집·recording rule·알림 구현
- [[Resilience4j]] — 서킷브레이커·레이트리미터로 에러 버짓 보호
- [[Logging-Strategy]] — 버짓 소진 원인 추적(구조화 로그·상관관계)
