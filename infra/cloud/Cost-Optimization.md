---
tags:
  - cloud
  - cost
  - finops
  - right-sizing
  - autoscaling
created: 2026-06-17
---

# 클라우드 비용 최적화 & Right-Sizing

> [!summary] 한 줄 요약
> 클라우드 비용은 **컴퓨트·스토리지·데이터 전송**이 핵심. 워크로드 특성에 맞는 구매 옵션(On-Demand/Reserved/Spot), 정확한 **right-sizing**(K8s requests/limits, 오토스케일링), 스토리지 계층화, egress 절감으로 낭비를 줄인다. 기술뿐 아니라 **태깅·예산 알림·FinOps 문화**가 지속적 최적화의 기반이다.

---

## 1. 비용 구조 이해
"어디에 돈이 새는지" 먼저 가시화해야 최적화가 가능하다.

```
[일반적 클라우드 비용 비중 (서비스마다 다름)]
  컴퓨트 (EC2/EKS/Lambda)   ~50-60%  ← 최대 절감 여지
  스토리지 (EBS/S3/RDS)     ~15-25%
  데이터 전송 (egress)       ~10-20%  ← 의외로 큼, 숨겨진 비용
  관리형 서비스 (RDS/ELB 등)  나머지
```

- 비용 = **단가 × 사용량 × 시간**. 셋 모두 최적화 대상.
- **유휴 자원(idle)** 이 가장 흔한 낭비: 끄지 않은 dev 환경, 미연결 EBS, 미사용 Elastic IP.

---

## 2. 컴퓨트 구매 옵션
워크로드의 **예측 가능성·중단 허용 여부**에 따라 선택.

| 옵션 | 할인 | 약정/제약 | 적합 워크로드 |
|------|------|-----------|---------------|
| **On-Demand** | 0% (기준가) | 없음 | 단기·예측 불가·테스트 |
| **Reserved (RI)** | ~40-60% | 1~3년 약정 | 상시 가동 베이스라인 |
| **Savings Plan** | ~40-65% | 시간당 $ 약정(유연) | 인스턴스 변경 잦은 상시 |
| **Spot** | ~70-90% | 언제든 회수(중단) 가능 | 무상태·배치·재시도 가능 |

- 전략: **베이스라인은 RI/Savings Plan, 변동분은 On-Demand, 내고장성 워크로드는 Spot** 혼합.
- Spot: [[Kubernetes]] 노드그룹에 섞어 쓰되 중단 핸들링(노드 drain) 필수. 배치/추론 워커에 적합.

---

## 3. 오토스케일링 임계 설정
스케일링은 비용과 성능의 핵심 레버. 임계가 잘못되면 과대 또는 과소.

```yaml
# HPA — CPU/메모리 또는 커스텀 메트릭 기반
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 3              # 가용성 하한 (피크 외에도 유지)
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # 너무 높으면 지연, 너무 낮으면 과대 프로비저닝
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 급격한 축소로 인한 플래핑 방지
```

- **scale-up은 빠르게, scale-down은 보수적으로** → 트래픽 출렁임에 안정적.
- 목표 사용률 60~70%가 일반적 출발점. 부하 테스트로 검증.
- 자세한 메커니즘은 [[HPA]] 참고.

---

## 4. K8s requests/limits Right-Sizing
**과대할당(over-provisioning)** 이 컨테이너 환경 최대 낭비원.

- `requests`: 스케줄러가 노드 자원을 예약하는 기준 → **너무 크면 노드에 적게 적재되어 낭비**.
- `limits`: 상한. CPU limit은 throttling, 메모리 limit 초과는 OOMKill.

```yaml
resources:
  requests:           # 실측 P50~P90 기반으로 설정
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m        # 버스트 허용 (CPU는 압축 가능 자원)
    memory: 512Mi     # 메모리는 requests에 근접하게 (OOM 주의)
```

| 도구 | 역할 |
|------|------|
| **VPA** | 실사용 기반으로 requests/limits 권장·자동 조정 (수직) |
| **HPA** | 부하에 따라 파드 수 조정 (수평) |
| Goldilocks / kube-resource-report | requests 대비 실사용 시각화 → right-sizing 근거 |

- VPA(권장 모드)로 적정값을 산출하고, HPA로 수평 확장 → **VPA + HPA는 CPU/메모리 메트릭에서 충돌**하므로 커스텀 메트릭이나 VPA recommend-only로 분리.

---

## 5. 스토리지 계층화
접근 빈도에 따라 저렴한 계층으로 자동 이동.

```
[S3 라이프사이클]
  Standard          → 자주 접근 (핫)
  Standard-IA       → 30일+ 비접근 (저렴, 검색비 발생)
  Glacier Instant   → 분기성 접근
  Glacier Deep      → 아카이브 (가장 저렴, 복원 수시간)
  → Lifecycle 정책으로 자동 전환 + 만료 삭제
```

- **EBS**: gp3가 gp2 대비 저렴 + IOPS 독립 설정. 미사용 볼륨/오래된 스냅샷 정리.
- **로그/메트릭**: 보존 기간 정책 필수 (무한 보존은 비용 폭증).

---

## 6. 데이터 전송(egress) 비용
숨겨진 비용 1순위. **나가는(egress) 트래픽**은 유료, 들어오는(ingress)은 보통 무료.

- **AZ 간 전송**도 과금 → 같은 AZ 내 통신 우선, 토폴로지 인식 라우팅.
- 인터넷 egress가 비쌈 → **CDN(CloudFront 등)** 으로 캐싱해 오리진 egress 절감.
- 리전 간 복제·교차 리전 호출 최소화.
- VPC Endpoint/PrivateLink로 NAT Gateway egress 우회(S3/DynamoDB는 Gateway Endpoint 무료).

---

## 7. 비용 할당 / 태깅
"누가/무엇이 쓰는가"를 모르면 최적화 책임 주체가 없다.

```
[필수 태그 정책]
  team / owner       — 책임 주체
  service / app      — 서비스 단위 집계
  env                — prod / staging / dev
  cost-center        — 회계 배분
```

- 태그 미부착 자원은 생성 차단(정책)하거나 정기 리포트로 적발.
- 태그 기반으로 팀별/서비스별 비용 대시보드 → **쇼백(showback)/차지백(chargeback)**.

---

## 8. 예산 알림 & FinOps 문화
- **예산 알림**: AWS Budgets 등으로 임계(예: 예상치 80%) 초과 시 Slack/메일 알림. 이상 급증(anomaly detection) 알림 병행.
- **FinOps**: 엔지니어·재무·경영이 함께 비용을 **지속적으로** 관리하는 문화.
  - 비용을 **엔지니어링 KPI**로 가시화 (PR/배포에 비용 영향 인지).
  - Inform(가시화) → Optimize(최적화) → Operate(거버넌스) 반복 사이클.
  - 비용 절감을 일회성 프로젝트가 아닌 일상 운영으로.

---

## 9. GPU 인스턴스 비용 (추론 서빙)
GPU는 시간당 단가가 매우 높아 별도 전략이 필요하다.

- **유휴 GPU = 막대한 낭비**: 추론 트래픽이 간헐적이면 GPU가 놀기 쉽다.
- 절감 수단:
  - **배치 추론 / 동적 배칭**으로 GPU 이용률 ↑.
  - **모델 양자화·경량화**로 작은 GPU(또는 CPU)에 적재.
  - 트래픽 없을 때 **scale-to-zero**(서버리스 추론, KEDA 기반 스케일링).
  - 학습/배치는 **Spot GPU**, 상시 추론은 RI로 분리.
- 자체 GPU 서빙 vs 매니지드/API 비용 분기점은 [[../../ai/GPU-Inference-Decision-Guide]] 참고.

| 선택 | 비용 특성 | 적합 |
|------|-----------|------|
| 전용 GPU 인스턴스 | 고정비 큼, 고이용률 시 유리 | 지속적 고트래픽 추론 |
| 서버리스/scale-to-zero | 사용량 기반, cold start | 간헐적 추론 |
| 외부 LLM API | 토큰당 과금, 운영부담 0 | 저~중 볼륨, 빠른 출시 |

---

## 10. 관련
- [[Kubernetes]] · [[HPA]] · [[../../ai/GPU-Inference-Decision-Guide]]
