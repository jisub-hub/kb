---
tags:
  - ai
  - yolo
  - object-detection
  - finetuning
  - computer-vision
  - training
created: 2026-06-17
---

# YOLO 파인튜닝 — 커스텀 객체 탐지 모델 학습

> [!summary] 한 줄 요약
> COCO 등으로 사전학습된 YOLO 가중치에서 출발해, 내 도메인 데이터로 재학습하는 것. 핵심은 **모델 튜닝이 아니라 데이터 품질·분포·증강**이다. 적은 데이터(수천 장)로도 전이학습 덕에 실용 모델이 나온다. 추론 배포는 [[DJL]]·[[../hardware/edge-ai/Jetson|Jetson]] 참고.

---

## 1. 파인튜닝이란 — 전이학습 기반

```
COCO pretrained YOLO (80클래스, 수십만 장 학습됨)
        │  내 데이터(중장비 등)로 재학습
        ▼
커스텀 탐지 모델 (내 클래스)

→ 처음부터 학습(scratch)하지 않는다. backbone의 일반 특징(엣지/질감/형태)을
  재활용하고, head를 내 클래스에 맞춰 적응시킨다.
→ 그래서 수천 장으로도 동작. scratch면 수십만 장 필요.
```

언제 파인튜닝: COCO에 없는 클래스(중장비, 결함, 특수 객체), 도메인 특화 환경(공사현장·CCTV·항공).

---

## 2. 데이터 준비

### 2.1 YOLO 라벨 포맷
```
이미지마다 같은 이름의 .txt 라벨:
  images/img001.jpg  ↔  labels/img001.txt

img001.txt (한 줄 = 한 객체, 정규화 0~1):
  <class_id> <x_center> <y_center> <width> <height>
  0 0.512 0.473 0.084 0.156    # class 0, 중심(x,y), 크기(w,h)
```

### 2.2 디렉토리 구조 & data.yaml
```
dataset/
  images/{train,val,test}/
  labels/{train,val,test}/
```
```yaml
# data.yaml
path: /data/excavator
train: images/train
val: images/val
test: images/test
names:
  0: excavator      # 굴착기
  1: dump_truck     # 덤프트럭
  2: crane          # 크레인
  3: loader         # 로더
```

### 2.3 분할 원칙
```
train : val : test ≈ 70 : 20 : 10
- val/test는 train과 "다른 현장·다른 날·다른 각도"로 (데이터 누수 방지)
- 같은 영상의 연속 프레임을 train/val에 쪼개 넣지 말 것 (거의 동일 → 과적합 착시)
```

---

## 3. 데이터 품질·분포 진단 (가장 중요, 학습 전 필수)

> 모델 성능의 80%는 데이터에서 결정된다. 학습 돌리기 전에 분포부터 본다.

```python
# bbox 면적 분포 — "스케일 편향" 진단
import numpy as np, matplotlib.pyplot as plt
areas = []  # 각 라벨의 w*h (정규화 면적)
for txt in label_files:
    for line in open(txt):
        _, _, _, w, h = map(float, line.split())
        areas.append(w * h)
plt.hist(np.sqrt(areas), bins=50)  # √면적 = 상대 크기
plt.title("객체 크기 분포")  # 작은 쪽에 쏠렸으면 스케일 편향
```

체크리스트:
```
□ 클래스별 인스턴스 수 (불균형 → 적은 클래스 성능 저하)
□ bbox 크기 분포 (작은 것에 쏠림 = 큰 객체 미탐지 원인)
□ 클래스별 다양한 각도·조명·배경이 있는가
□ 라벨 정확도 (잘못된/누락 박스는 독)
□ train/val 분포가 비슷한가
```

---

## 4. Augmentation (증강) — 적은 데이터의 핵심

| 증강 | 효과 | 파라미터(Ultralytics) | 주의 |
|------|------|----------------------|------|
| **Mosaic** | 4장 합성 → 스케일·맥락 다양화, 작은 객체↑ | `mosaic: 1.0` | 큰 객체엔 불리할 수 있음 |
| **Close Mosaic** | 마지막 N epoch mosaic 끔 → 안정화 | `close_mosaic: 10` | 큰 객체 학습 회복 |
| **Scale jitter** | 이미지 축소/확대 → 스케일 분산 | `scale: 0.5~0.9` | 스케일 편향 교정 |
| **Copy-Paste** | 객체 복사-붙여넣기 → 개수·크기↑ | `copy_paste: 0.3` | 작은/희소 클래스에 유효 |
| **MixUp** | 두 이미지 블렌딩 | `mixup: 0.1` | 과하면 라벨 노이즈 |
| **HSV** | 색상·명도 변화 | `hsv_h/s/v` | 조명 변화 대응 |
| Flip/Rotate | 좌우반전 등 | `fliplr: 0.5` | 방향성 있는 객체 주의 |

```
원칙: 증강은 "테스트 환경에서 실제로 나타날 변화"를 모사해야 한다.
  공사현장 = 다양한 거리(scale), 조명(HSV), 먼지/가림(occlusion) → 해당 증강 강화
```

---

## 5. 학습 설정

### 5.1 모델 크기 선택
```
yolov8n / yolo11n  — nano, 가장 빠름, 엣지(Jetson), 정확도↓
       s           — small, 균형 (실무 기본 출발점)
       m           — medium, 정확도↑, 속도↓
       l / x       — large/xlarge, 최고 정확도, 무거움

→ 작은 객체·고정확 필요 = m 이상. 엣지 실시간 = n/s.
→ 데이터 3700장 정도면 s~m에서 시작, 과적합 보며 조정.
```

### 5.2 핵심 하이퍼파라미터
```
epochs: 100~300       # early stopping(patience)과 함께
imgsz: 640 → 1280     # 작은 객체면 키운다 (VRAM·시간 trade)
batch: GPU에 맞게 (-1 = auto)
optimizer: auto (SGD/AdamW)
lr0: 0.01 (SGD) / 0.001 (AdamW)
patience: 50          # val 개선 없으면 조기 종료
pretrained: True      # 전이학습 (필수)
```

---

## 6. 작은 객체 & 스케일 분산 대응 ⭐ (현장 핵심)

> 공사현장처럼 **표적이 멀어 작고**, 가까이 오면 커지는 환경의 정석 대응.

### 문제 분리
```
A. 작은 표적(멀리) 미탐지   → small object detection 난제
B. 큰 표적(가까이) 미탐지   → 학습 데이터가 작은 것에 쏠린 "스케일 편향"
```
**B가 의외로 흔하고 근본적**이다 — 작은 것만 학습하면 큰 것을 모른다. 모델 한계가 아니라 데이터 분포 문제.

### 대응 (효과 순)
| 순서 | 조치 | 해결 | 방법 |
|------|------|------|------|
| 1 | bbox 면적 분포 진단 | 진단 | 3절 히스토그램 |
| 2 | **가까운/큰 객체 데이터 보강** | B | 수집·라벨링 |
| 3 | scale·mosaic·copy_paste 증강 | A+B | 4절 |
| 4 | **고해상도 학습** imgsz 1280~1536 | A | 작은 객체 픽셀↑ |
| 5 | **SAHI 슬라이스 추론** | A | 아래 |
| 6 | P2 head / 큰 모델 | A | 구조 변경 |

### SAHI (Slicing Aided Hyper Inference)
```python
# 큰 이미지를 타일로 쪼개 추론 → 작은 객체가 타일 안에서 "커져" 탐지율 급상승
# 원거리 광각 공사현장 영상에 특히 효과적
from sahi import AutoDetectionModel
from sahi.predict import get_sliced_prediction

model = AutoDetectionModel.from_pretrained(
    model_type="ultralytics", model_path="best.pt", confidence_threshold=0.3)
result = get_sliced_prediction(
    "site.jpg", model,
    slice_height=640, slice_width=640,
    overlap_height_ratio=0.2, overlap_width_ratio=0.2)
```

### P2 detection head
```
기본 YOLO 헤드: P3/P4/P5 (stride 8/16/32)
작은 객체용 P2 헤드 추가(stride 4) → 고해상도 특징으로 작은 객체 mAP↑
  yolov8-p2.yaml / yolo11-p2.yaml 변형 사용 (연산량↑)
```

---

## 7. 학습 실행 (Ultralytics)

```bash
# CLI
yolo detect train data=data.yaml model=yolo11s.pt \
  epochs=200 imgsz=1280 batch=16 \
  scale=0.7 mosaic=1.0 copy_paste=0.3 close_mosaic=10 \
  patience=50 project=excavator name=v1
```
```python
# Python API
from ultralytics import YOLO
model = YOLO("yolo11s.pt")          # pretrained
model.train(
    data="data.yaml", epochs=200, imgsz=1280, batch=16,
    scale=0.7, mosaic=1.0, copy_paste=0.3, close_mosaic=10,
    patience=50, project="excavator", name="v1")
metrics = model.val()               # val 평가
```

---

## 8. 평가 — 무엇을 보나

| 지표 | 의미 | 주의 |
|------|------|------|
| **mAP50** | IoU 0.5 기준 평균 정밀도 | 너무 관대, 기본 지표 |
| **mAP50-95** | IoU 0.5~0.95 평균 | 더 엄격, 박스 정확도 반영 |
| Precision | 예측 중 맞은 비율 | 오탐(false positive) 척도 |
| Recall | 실제 중 잡은 비율 | 미탐(false negative) 척도 |
| **per-class mAP** | 클래스별 성능 | 불균형·약한 클래스 발견 |
| Confusion Matrix | 클래스 혼동·배경 오탐 | 어떤 클래스끼리 헷갈리는지 |

```
작은 객체 문제면 → mAP를 객체 크기별(small/medium/large)로 분해해 본다.
  small의 mAP만 낮으면 A 문제(해상도·SAHI),
  large의 mAP만 낮으면 B 문제(데이터 편향).
```

> 지표 상세(IoU→Precision/Recall→AP→mAP, @50 vs @50-95, 크기별 진단) → [[Object-Detection-Metrics]]

---

## 9. 흔한 문제 & 디버깅

| 증상 | 원인 | 대응 |
|------|------|------|
| 큰 객체 미탐지 | 학습 데이터 스케일 편향(B) | 큰 객체 데이터·scale 증강 |
| 작은 객체 미탐지 | 해상도 부족(A) | imgsz↑, SAHI, P2 |
| train↓ val↑ (과적합) | 데이터 적음·증강 부족 | 증강↑, 데이터↑, 작은 모델 |
| 특정 클래스만 낮음 | 클래스 불균형 | 해당 클래스 데이터·copy_paste |
| 배경을 객체로 오탐 | hard negative 부족 | 객체 없는 배경 이미지 추가 |
| 박스는 맞는데 위치 부정확 | mAP50-95 낮음 | 라벨 품질, 더 큰 모델 |

---

## 10. 배포 연계

```bash
# 학습한 모델을 추론용으로 export
yolo export model=best.pt format=onnx        # ONNX (범용)
yolo export model=best.pt format=engine      # TensorRT (NVIDIA 가속)
yolo export model=best.pt format=coreml      # Apple
```
- Java 서비스 추론: [[DJL]] (ONNX 로드, CCTV 연동)
- 엣지 실시간: [[../hardware/edge-ai/Jetson]] (TensorRT, DeepStream 다채널)
- 추론 최적화 원리: [[Inference-Optimization]] (양자화 INT8 등)

---

## 11. 현장 케이스 — 중장비 식별 (3700장, 원거리)

```
상황: 공사현장, 표적이 멀어 작음. 가까워지면(커지면) 미탐지. 라벨 3700장.

진단: 학습 데이터가 "멀리 있는 작은 표적"에 쏠림 → 큰 객체 분포 부재(B)
      + 작은 객체 자체의 해상도 난제(A)

실행 순서:
  1. bbox 면적 분포 확인 → 작은 쪽 쏠림 정량 확인
  2. 가까운/큰 중장비 프레임 수집·라벨링 (분포 채우기) ← B 직격
  3. scale=0.7, mosaic, copy_paste=0.3로 스케일 다양화
  4. imgsz=1280으로 학습 (작은 객체 픽셀 확보)
  5. 추론에 SAHI 적용 (원거리 작은 표적 탐지율↑)
  6. 부족하면 yolo11s→m, P2 head 검토
  7. 크기별 mAP로 A/B 어느 쪽이 개선됐는지 추적
```

> 핵심: **큰 객체 미탐지는 모델이 아니라 데이터 편향**이다. 모델만 키우지 말고 데이터 분포부터 채워라.

---

## 관련
- [[YOLO-Detection-Pipeline]] — 동작 원리(그리드·NMS·loss) + cascade 파이프라인(사람→머리crop→안전모)
- [[DJL]] — 학습된 YOLO를 Java로 추론 배포 (ONNX, CCTV)
- [[../hardware/edge-ai/Jetson]] — 엣지 실시간 추론 (TensorRT/DeepStream)
- [[Fine-Tuning]] — LLM 파인튜닝 (LoRA/QLoRA) — 개념 대비
- [[Inference-Optimization]] — 추론 양자화·가속
- [[ML-Fundamentals]] — 과적합, train/val/test, 평가 지표
- [[VLM]] — Vision Language Model (탐지 대신 이해)
