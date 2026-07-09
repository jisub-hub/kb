---
tags:
  - deploy
  - feature-flags
  - release
  - spring
  - experimentation
created: 2026-06-17
---

# Feature Flags (기능 플래그 / 토글)

> [!summary] 한 줄 요약
> 코드 배포와 기능 출시를 **분리**하는 기법. 배포된 코드의 기능을 런타임 스위치로 켜고 끈다 → 다크 런치, 점진 롤아웃, 즉시 롤백(재배포 없이), A/B 실험이 가능. [[Deployment-Strategies]]의 카나리와 짝을 이룬다.

---

## 1. 왜 쓰는가 — 배포 ≠ 출시

```
플래그 없음: 코드 머지 = 즉시 모든 사용자에게 노출
  → 문제 시 롤백 = 재배포(느림), 미완성 기능 머지 불가(장수 브랜치)

플래그 있음: 코드는 배포하되 기능은 OFF로 숨김
  → 트렁크에 미완성 기능 머지 가능(점진 개발)
  → 출시 = 플래그 ON (재배포 없이 즉시)
  → 문제 = 플래그 OFF (즉시 롤백, 재배포 불필요)
  → 일부 사용자에게만 ON (점진 롤아웃·A/B)
```

[[Git-Workflow]]의 트렁크 기반 개발과 결합해 **장수 브랜치 없이** 미완성 기능을 안전하게 통합한다.

---

## 2. 토글의 4가지 종류 (수명·변경 빈도가 다름)

| 종류 | 목적 | 수명 | 예 |
|------|------|------|-----|
| **Release** | 미완성/신기능 숨김 | 짧음(출시 후 제거) | 신규 결제 UI |
| **Ops** | 운영 중 기능 on/off (킬 스위치) | 길음 | 부하 시 무거운 추천 끄기 |
| **Experiment** | A/B 테스트 | 실험 기간 | 버튼 색·알고리즘 비교 |
| **Permission** | 사용자군별 기능 | 영구 | 프리미엄/베타 사용자 |

> 종류를 구분해야 **관리·제거 정책**이 달라진다. Release 토글을 안 지우면 부채가 된다(7절).

---

## 3. 점진 롤아웃 & 카나리 연계

```
0% → 내부 직원만 → 1% → 10% → 50% → 100%
  각 단계에서 메트릭([[SLO-SLI]]) 관찰 → 이상 시 즉시 0%로

vs 카나리 배포([[Deployment-Strategies]]):
  카나리 = "새 버전 인스턴스"에 트래픽 일부 (인프라 레벨)
  플래그 = "기능"을 사용자 일부에게 (애플리케이션 레벨, 더 세밀)
  → 둘을 함께: 카나리로 새 코드 배포 + 플래그로 기능 점진 노출
```

대상 지정(targeting): 사용자 ID 해시(%), 사용자 속성(국가·등급), 그룹.

---

## 4. Spring 구현

```java
// 간단한 자체 구현 — 설정/DB 기반
@Component
@RequiredArgsConstructor
public class FeatureFlags {
    private final FlagRepository repo;          // DB/Redis/설정에서 조회

    public boolean isEnabled(String key, String userId) {
        Flag f = repo.find(key);
        if (f == null || !f.isOn()) return false;
        // 점진 롤아웃: userId 해시 % 100 < rolloutPercent
        return Math.floorMod(userId.hashCode(), 100) < f.getRolloutPercent();
    }
}

// 사용
if (featureFlags.isEnabled("new-checkout", userId)) {
    return newCheckout(...);   // 신 기능
} else {
    return legacyCheckout(...); // 기존
}
```

```yaml
# Spring Cloud Config / @ConfigurationProperties 로 외부화
features:
  new-checkout: { on: true, rolloutPercent: 10 }
```

> 핵심: 플래그 평가는 **빠르고(캐시) 안전한 기본값**을 가져야 한다. 조회 실패 시 OFF(보수적)로 폴백.

---

## 5. 관리 도구

| 도구 | 특징 |
|------|------|
| **Unleash** | 오픈소스, self-host, Spring SDK, 전략(점진/그룹) |
| **GrowthBook** | 오픈소스, A/B 실험·통계 분석 강점 |
| **LaunchDarkly** | 상용 SaaS, 엔터프라이즈 기능 풍부 |
| Flagsmith / 자체 구현 | 소규모는 DB/Config로 충분 |

> 실험(A/B)까지 하려면 GrowthBook/LaunchDarkly, 단순 릴리스/킬스위치는 자체 구현이나 Unleash로 충분.

---

## 6. A/B 테스트 연계

```
플래그로 사용자를 A군(기존)/B군(신규)으로 분기
  → 각 군의 지표(전환율·체류·오류) 수집
  → 통계적 유의성 검정 후 승자 채택
주의: 같은 사용자는 항상 같은 군(sticky) — userId 해시 고정
```

→ LLM 응답 A/B 등은 [[../../../ai/LLMOps]]의 모델 버전 비교와도 연결.

---

## 7. 기술 부채 관리 (가장 흔한 함정)

```
⚠️ Release 토글을 출시 후 안 지움 → if/else가 코드에 쌓임
  → 조합 폭발(플래그 N개 = 2^N 경로), 테스트 불가, 죽은 코드

관리 원칙:
  - 플래그마다 "만료일/제거 조건" 명시 (소유자 지정)
  - 출시 완료된 Release 토글은 정리 티켓 생성 → 코드에서 제거
  - 플래그 인벤토리 주기 점검 (오래된 토글 알림)
  - Ops/Permission 토글은 영구지만 Release는 임시임을 구분
```

---

## 8. 정리

```
배포(코드 전달) ≠ 출시(기능 노출) — 플래그가 둘을 분리
효과: 다크 런치 · 점진 롤아웃 · 즉시 롤백(무재배포) · A/B · 킬 스위치
종류: Release(임시)/Ops/Experiment/Permission — 수명이 다름
함정: Release 토글 미제거 → 부채. 만료·소유자·정리 정책 필수
```

> 플래그는 강력하지만 공짜가 아니다 — **분기 복잡도와 정리 부채**를 낸다. "이 플래그는 언제 제거되나"를 만들 때 정해야 한다.

---

## 관련
- [[Deployment-Strategies]] — 카나리/Blue-Green (인프라 레벨), 플래그(앱 레벨)와 상보
- [[Git-Workflow]] — 트렁크 기반 개발 + 플래그로 미완성 기능 통합
- [[CICD]] — 배포 파이프라인과 플래그 분리
- [[SLO-SLI]] — 점진 롤아웃 중 메트릭 관찰
- [[../../../ai/LLMOps]] — 모델 A/B 테스트
