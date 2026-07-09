---
tags:
  - ai
  - yolo
  - object-detection
  - computer-vision
  - pipeline
  - architecture
created: 2026-06-17
---

# YOLO 동작 원리 & 탐지 파이프라인

> [!summary] 한 줄 요약
> YOLO는 이미지를 한 번의 forward pass로 그리드 위에서 **"객체 유무(objectness) + 위치(박스 회귀) + 클래스"**를 동시에 예측하고 NMS로 정리한다. 작은 객체나 정밀 판별이 필요하면 **cascade 파이프라인**(예: 사람 탐지 → 머리 crop → 안전모 탐지)으로 정밀도를 끌어올린다. 학습/파인튜닝은 [[YOLO-Finetuning]], 배포는 [[DJL]]·[[../hardware/edge-ai/Jetson|Jetson]].

---

# Part 1. 동작 원리 (Deep Dive)

## 1. 전체 아키텍처 — Backbone / Neck / Head

```
이미지 (640×640×3)
   │
   ▼
[Backbone]  특징 추출 (CSPDarknet 등)
   │  점점 해상도↓, 채널(의미)↑ → 계층적 특징
   ▼
[Neck]  멀티스케일 특징 융합 (FPN + PAN)
   │  깊은 층(의미)과 얕은 층(위치)을 결합
   ▼
[Head]  그리드별 예측 (박스 + objectness + 클래스)
   │
   ▼
[NMS]  중복 박스 제거 → 최종 탐지
```

- **Backbone**: 이미지를 점점 줄여가며 엣지→질감→부품→객체 형태의 계층적 특징을 뽑는다. (전이학습으로 재사용되는 부분)
- **Neck (FPN+PAN)**: 깊은 층은 "무엇인지"(의미)에 강하지만 위치가 뭉개지고, 얕은 층은 "어디"(위치)에 강하지만 의미가 약하다. Neck이 둘을 양방향으로 섞어 각 스케일이 의미+위치를 모두 갖게 한다.
- **Head**: 실제 박스/클래스를 출력하는 예측 층.

## 2. 그리드 & 예측 텐서

이미지는 stride에 따라 그리드로 나뉜다. 입력 640, stride 8 → 80×80 그리드.

```
각 그리드 위치(앵커)가 예측하는 벡터:
[ tx, ty, tw, th,  obj,  c1, c2, ... cn ]
  └── 박스 ──┘   └객체?┘  └─ 클래스 확률 ─┘

obj (objectness) : "여기 객체가 있을 확률" (배경 거르기)
최종 신뢰도        = obj × max(class 확률)
```

**책임 분배**: 객체의 *중심*이 떨어진 그리드 위치가 그 객체를 예측할 책임을 진다.

## 3. 박스 좌표 인코딩/디코딩 (회귀의 핵심)

모델은 절대 좌표를 직접 뱉지 않는다. 그리드 셀 기준 **상대값(offset)**을 예측하고, 디코딩으로 픽셀 좌표를 복원한다.

```
앵커 기반 (YOLOv5 등):
  bx = (σ(tx) + cx)          # cx,cy = 셀 위치
  by = (σ(ty) + cy)
  bw = anchor_w · e^(tw)     # 사전 정의 앵커 크기에 배율
  bh = anchor_h · e^(th)
  → "기준 앵커에서 얼마나 벗어났나"만 학습 → 안정적

앵커 프리 (YOLOv8/v11/X):
  중심점에서 네 변까지 거리(l,t,r,b)를 직접 회귀
  앵커 하이퍼파라미터 불필요 → 단순, 최신 추세
```

> 왜 offset만 학습하나: 절대 좌표(0~640)를 직접 회귀하면 분산이 커 학습이 불안정하다. "셀 기준 작은 보정값"으로 바꾸면 학습이 쉬워진다.

### DFL (Distribution Focal Loss) — 최신 박스 회귀
```
v8+: 좌표를 단일 숫자가 아니라 "이산 분포"로 예측
  예: 변까지 거리 = Σ(i × softmax 확률)
  → 경계가 모호한 객체(가림·흐림)에서 더 정확한 박스
```

## 4. 멀티스케일 — 작은 것과 큰 것을 다른 층이 담당 ⭐

```
Head는 여러 해상도 feature map에서 동시에 예측:
  P3 (stride 8,  80×80)  → 작은 객체 (멀리 있는 표적, 안전모)
  P4 (stride 16, 40×40)  → 중간 객체
  P5 (stride 32, 20×20)  → 큰 객체 (가까운 사람·차량)
  (+ P2 stride 4 추가 시 더 작은 객체)

→ 작은 표적과 큰 표적을 서로 다른 층이 나눠 담당
→ 한 층이 모든 크기를 감당하지 않아 정확도↑
```

이것이 [[YOLO-Finetuning]]에서 "작은 객체엔 P2 head·고해상도(imgsz↑)"가 통하는 이유다.

## 5. Label Assignment — 학습 시 "누가 이 객체를 맡나"

학습의 숨은 핵심. 한 GT(정답 박스)에 어떤 예측을 양성(positive)으로 매칭할지 정하는 규칙.

```
구식: GT 중심 셀 + IoU 높은 앵커 1개만 양성
최신(v8): Task-Aligned Assigner
  → classification 점수 × localization(IoU)를 함께 보고
    "분류도 잘하고 박스도 잘 맞는" 예측을 양성으로 동적 선택
  → 더 빠른 수렴, 더 높은 정확도
(YOLOX: SimOTA — 최적 수송 기반 동적 할당)
```

> 같은 데이터·모델이어도 assignment 전략이 성능을 크게 가른다.

## 6. Loss — 세 가지를 합쳐 끌어당김

```
Total Loss = λ_box · Box Loss      (CIoU/DIoU — 박스 겹침·중심·종횡비)
           + λ_obj · Obj Loss      (BCE — 객체 있는 곳 1, 배경 0)
           + λ_cls · Cls Loss      (BCE — 올바른 클래스 확률↑)
           + λ_dfl · DFL           (분포 기반 박스 정밀도, v8+)

→ 역전파로 backbone~head 가중치 업데이트
```

## 7. NMS — 중복 제거 (후처리)

한 객체를 여러 셀·스케일이 동시에 예측하므로 중복이 생긴다.

```
NMS (Non-Maximum Suppression):
  1. 신뢰도 내림차순 정렬
  2. 1위 박스 채택 → 그와 IoU > 임계(예 0.45)인 박스 제거
  3. 남은 것 중 다시 1위 채택 → 반복
  → 객체당 박스 1개

변형:
  Soft-NMS  : 제거 대신 점수 감쇠 (겹친 객체 보존)
  클래스별 NMS: 클래스 단위로 따로 (다른 클래스 겹침 허용)
  NMS-free  : 일부 최신 모델(YOLOv10)은 NMS 제거 (end-to-end)
```

## 8. 추론 흐름 한눈에

```
이미지 → 전처리(resize/letterbox/정규화) → Backbone → Neck
      → Head(그리드별 raw 예측) → 디코딩(offset→픽셀)
      → confidence 필터(obj×cls > thresh) → NMS → 최종 박스
```

---

# Part 2. 탐지 파이프라인 — Single-stage vs Cascade

## 9. 왜 단일 모델로 부족할 때가 있나

단일 YOLO로 `[사람, 안전모, 미착용]`을 한 번에 탐지할 수 있다. 하지만:

```
문제: 안전모는 전체 프레임에서 "작은 객체"
  → 멀리 있는 작업자의 안전모는 몇 픽셀 → 직접 탐지 시 놓침/오탐
  → "누가" 썼는지 연관(association)도 약함
  → 배경의 비슷한 물체(빨간 양동이 등)를 안전모로 오탐
```

## 10. Cascade 파이프라인 — 사람 → 머리 crop → 안전모 ⭐

```
[1단계] 사람 탐지 (YOLO person model)
   → 프레임에서 사람 bbox 추출

[2단계] 각 사람 bbox에서 머리 영역 crop
   → 사람 박스 상단 ~25~30%를 잘라냄 (또는 pose로 머리 키포인트)

[3단계] crop에 안전모 탐지/분류
   → 작게 잘린 머리 영역에선 안전모가 "상대적으로 크다"
   → helmet / no-helmet 판별

[4단계] 결과 결합
   → 사람별 "안전모 착용 여부" 라벨
```

```python
# 개념 코드 (Ultralytics)
person_res = person_model(frame, classes=[0])   # 사람만
for box in person_res[0].boxes:
    x1, y1, x2, y2 = map(int, box.xyxy[0])
    h = y2 - y1
    head = frame[y1 : y1 + int(h * 0.30), x1:x2]  # 상단 30% 머리 영역
    if head.size == 0:
        continue
    helmet_res = helmet_model(head)               # 2차 추론(작은 영역)
    worn = len(helmet_res[0].boxes) > 0           # 안전모 검출 여부
    label = "안전모 O" if worn else "⚠️ 미착용"
    # → 사람 박스(x1,y1,x2,y2)에 label 매핑 (연관 명확)
```

## 11. 왜 cascade가 정밀도를 높이나

| 효과 | 원리 |
|------|------|
| **작은 객체 → 큰 객체화** | 머리 영역을 crop·확대하면 안전모가 입력에서 커져 탐지 쉬움 (SAHI와 같은 원리) |
| **배경 false positive 감소** | 안전모를 "사람 머리 위"에서만 찾음 → 배경의 유사물 오탐 제거 |
| **연관(association) 명확** | "어느 사람이" 착용/미착용인지 1:1 매핑 (단일 모델은 누구 것인지 모호) |
| **2차 모델 단순화** | 좁은 도메인(머리 영역)만 보므로 작은 모델로도 고정밀 |

## 12. 파이프라인 변형·고도화

```
머리 위치 정밀화:
  사람 bbox 상단 crop  →  Pose Estimation(키포인트)으로 머리/어깨 추정 → 더 정확한 crop

영상(연속 프레임):
  + 객체 추적(ByteTrack/BoT-SORT) → 사람마다 ID 부여
  + 시간적 투표(여러 프레임 다수결) → 단발 오탐 억제 (한 프레임 놓쳐도 보정)

다단계 일반화 (PPE 전체):
  사람 → 부위별 crop → [안전모(머리) / 안전조끼(몸통) / 장갑(손)] 각각 판별

ROI 한정:
  위험구역(폴리곤) 안의 사람만 검사 → 연산 절감 + 정책 정렬
```

## 13. 단일 모델 vs Cascade — 선택 기준

| 항목 | 단일 멀티클래스 | Cascade (2-stage) |
|------|----------------|------------------|
| 속도 | 빠름 (1회 추론) | 느림 (사람 N명 → N회 2차 추론) |
| 작은 객체 정밀도 | 낮음 | **높음** (crop 확대) |
| 연관(누가 착용) | 모호 | **명확** |
| 구현 복잡도 | 낮음 | 높음 (2모델·crop·결합) |
| 적합 상황 | 객체가 충분히 큼·근거리 | 원거리·작은 PPE·정밀 요구 |

```
권장:
  근거리·큰 화면     → 단일 모델(person+helmet+no-helmet 클래스)이 충분
  원거리 공사현장    → cascade (사람 탐지 → 머리 crop → 안전모) + 트래킹
  실시간 다채널 엣지 → 단일 모델 우선, 정밀 필요 구역만 cascade
```

> 트레이드오프 본질: cascade는 **정밀도·연관을 사는 대신 지연·복잡도를 낸다.** 사람 수가 많은 프레임에서 2차 추론이 N배로 늘어나니, 엣지([[../hardware/edge-ai/Jetson|Jetson]])에선 배칭·ROI 한정으로 비용을 관리한다.

---

## 14. 정리

```
동작 원리: 그리드 × (objectness + 박스 회귀 + 클래스)를 단일 forward pass로
           → 멀티스케일 head가 크기별 분담 → NMS로 정리
           → 학습은 label assignment + (CIoU+BCE+DFL) loss

파이프라인: 작은 객체·정밀 판별 = cascade
           사람 탐지 → 머리 crop(확대) → 안전모 탐지 → 사람별 결합
           = 작은 객체를 키우고 + 배경 오탐 줄이고 + 연관 명확화
```

---

## 관련
- [[YOLO-Finetuning]] — 학습/파인튜닝, 작은 객체·스케일 편향 대응(P2/imgsz/SAHI)
- [[Object-Detection-Metrics]] — NMS·objectness 출력 평가: IoU·AP·mAP@50-95
- [[DJL]] — 학습한 YOLO를 Java로 추론 배포 (CCTV)
- [[../hardware/edge-ai/Jetson]] — 엣지 실시간 다채널 추론 (DeepStream)
- [[Inference-Optimization]] — 추론 양자화·가속 (TensorRT)
- [[VLM]] — 탐지 대신 "이해"가 필요할 때 (장면 설명·질의)
