---
tags:
  - ai
  - djl
  - java
  - object-detection
  - yolo
  - deep-learning
created: 2026-06-16
---

# DJL (Deep Java Library) — Java AI 추론

> [!summary] 한 줄 요약
> **DJL**은 Amazon이 개발한 Java 네이티브 딥러닝 라이브러리. PyTorch/TensorFlow/MXNet 모델을 JVM에서 직접 추론. CCTV·IoT 데이터에서 **YOLO 객체 감지**를 Spring Boot에 통합할 때 핵심 도구.

---

## 1. DJL 개요

```
개발: Amazon Web Services
라이선스: Apache 2.0
지원 엔진: PyTorch (주력), TensorFlow, MXNet, ONNX Runtime, TensorRT
Model Zoo: 사전 훈련 모델 100+ (YOLO, ResNet, BERT, Whisper 등)

특징:
  - Java API — Python 없이 JVM에서 직접 추론
  - Lazy Loading — 추론 시점에 네이티브 라이브러리 자동 다운로드
  - 자동 하드웨어 감지 — CUDA GPU 있으면 자동 GPU 사용
  - Spring Boot Starter 제공
  - JavaCV(FFmpeg 프레임)와 Mat → BufferedImage 변환으로 연동 가능
```

---

## 2. 의존성

```xml
<!-- pom.xml -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>ai.djl</groupId>
      <artifactId>bom</artifactId>
      <version>0.30.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- DJL Core API -->
  <dependency>
    <groupId>ai.djl</groupId>
    <artifactId>api</artifactId>
  </dependency>

  <!-- PyTorch 엔진 (CUDA 자동 감지) -->
  <dependency>
    <groupId>ai.djl.pytorch</groupId>
    <artifactId>pytorch-engine</artifactId>
  </dependency>
  <dependency>
    <groupId>ai.djl.pytorch</groupId>
    <artifactId>pytorch-native-auto</artifactId>  <!-- CPU/CUDA 자동 선택 -->
    <runtime>true</runtime>
  </dependency>

  <!-- ONNX Runtime (범용, 경량) -->
  <dependency>
    <groupId>ai.djl.onnxruntime</groupId>
    <artifactId>onnxruntime-engine</artifactId>
  </dependency>
  <dependency>
    <groupId>com.microsoft.onnxruntime</groupId>
    <artifactId>onnxruntime</artifactId>
    <version>1.18.0</version>
  </dependency>

  <!-- Computer Vision 유틸 -->
  <dependency>
    <groupId>ai.djl</groupId>
    <artifactId>basicdataset</artifactId>
  </dependency>

  <!-- Model Zoo (사전 훈련 모델) -->
  <dependency>
    <groupId>ai.djl.pytorch</groupId>
    <artifactId>pytorch-model-zoo</artifactId>
  </dependency>
</dependencies>
```

---

## 3. 핵심 API 구조

```
[추론 파이프라인]
  원시 입력 (이미지/텍스트)
       │
  Translator.processInput()   ← 전처리: 리사이즈, 정규화, 텐서 변환
       │
  [Model] → forward()         ← 실제 추론 (GPU/CPU)
       │
  Translator.processOutput()  ← 후처리: 소프트맥스, NMS, 결과 파싱
       │
  Output (DetectedObjects 등)

[주요 클래스]
  Model      — 로드된 모델 파일 (.pt, .onnx)
  Predictor  — 추론 실행기 (스레드 안전하지 않음 → 스레드당 1개)
  Translator — 입출력 변환 로직
  NDArray    — 다차원 텐서
  Image      — DJL 이미지 래퍼
```

---

## 4. 객체 감지 — DJL Model Zoo (사전 훈련)

```java
@Service
@Slf4j
public class ObjectDetectionService {

    private ZooModel<Image, DetectedObjects> model;

    @PostConstruct
    public void init() throws Exception {
        // DJL Model Zoo에서 SSD 사전 훈련 모델 자동 다운로드
        Criteria<Image, DetectedObjects> criteria = Criteria.builder()
            .optApplication(Application.CV.OBJECT_DETECTION)
            .setTypes(Image.class, DetectedObjects.class)
            .optFilter("backbone", "mobilenet1.0")   // SSD-MobileNet
            .optFilter("dataset", "coco")             // COCO 80 클래스
            .optEngine("PyTorch")
            .optProgress(new ProgressBar())
            .build();

        model = ModelZoo.loadModel(criteria);
        log.info("Model loaded: {}", model.getName());
    }

    // Predictor는 스레드 안전하지 않음 → ThreadLocal 사용
    private final ThreadLocal<Predictor<Image, DetectedObjects>> predictorHolder =
        ThreadLocal.withInitial(() -> model.newPredictor());

    public List<Detection> detect(BufferedImage image, float minConfidence) {
        Image djlImage = ImageFactory.getInstance().fromImage(image);
        try {
            DetectedObjects result = predictorHolder.get().predict(djlImage);
            return result.items().stream()
                .filter(item -> item.getProbability() >= minConfidence)
                .map(item -> Detection.builder()
                    .className(item.getClassName())
                    .confidence((float) item.getProbability())
                    .boundingBox(item.getBoundingBox())
                    .build())
                .toList();
        } catch (TranslateException e) {
            throw new RuntimeException("Detection failed", e);
        }
    }

    @PreDestroy
    public void close() {
        model.close();
    }
}
```

---

## 5. YOLO 커스텀 모델 로드 (ONNX)

### YOLO → ONNX 변환 (Python)

```python
# Python에서 한 번만 실행
from ultralytics import YOLO

model = YOLO("yolov8n.pt")
model.export(format="onnx", imgsz=640, opset=12)
# → yolov8n.onnx 생성
```

### DJL로 ONNX 모델 로드

```java
@Configuration
public class YoloModelConfig {

    @Bean
    @SneakyThrows
    public ZooModel<Image, DetectedObjects> yoloModel() {
        // 커스텀 Translator (YOLO 출력 형식 파싱)
        YoloV8Translator translator = YoloV8Translator.builder()
            .optThreshold(0.5f)
            .optNmsThreshold(0.45f)
            .optSynsetArtifactName("coco.names")  // 클래스 이름 파일
            .build();

        Criteria<Image, DetectedObjects> criteria = Criteria.builder()
            .setTypes(Image.class, DetectedObjects.class)
            .optModelPath(Paths.get("models/yolov8n.onnx"))
            .optTranslator(translator)
            .optEngine("OnnxRuntime")
            .optDevice(Device.gpu())      // GPU 사용 (없으면 CPU 폴백)
            .build();

        return ModelZoo.loadModel(criteria);
    }
}
```

### YOLO Translator 직접 구현

```java
public class YoloV8Translator implements Translator<Image, DetectedObjects> {

    private static final int INPUT_SIZE = 640;
    private final float confThreshold;
    private final float nmsThreshold;
    private List<String> classes;

    @Override
    public NDList processInput(TranslatorContext ctx, Image input) {
        NDManager manager = ctx.getNDManager();

        // 1. 이미지 리사이즈 + letterbox padding
        Image resized = input.resize(INPUT_SIZE, INPUT_SIZE, true);

        // 2. HWC → CHW, BGR → RGB, 정규화 [0,255] → [0,1]
        NDArray array = resized.toNDArray(manager, Image.Flag.COLOR);
        array = array.transpose(2, 0, 1);         // HWC → CHW
        array = array.toType(DataType.FLOAT32, false);
        array = array.div(255.0f);                 // 정규화
        array = array.expandDims(0);               // [1, 3, 640, 640]

        return new NDList(array);
    }

    @Override
    public DetectedObjects processOutput(TranslatorContext ctx, NDList list) {
        // YOLO v8 출력: [1, 84, 8400]
        // 84 = 4(bbox: cx,cy,w,h) + 80(COCO 클래스 확률)
        NDArray output = list.singletonOrThrow().squeeze(0); // [84, 8400]
        output = output.transpose();                          // [8400, 84]

        // cx, cy, w, h 좌표 추출
        NDArray boxes = output.get(":, 0:4");
        // 클래스 확률 추출
        NDArray scores = output.get(":, 4:");
        NDArray maxScores = scores.max(new int[]{1});
        NDArray classIds = scores.argMax(1);

        // 신뢰도 임계값 필터링
        NDArray mask = maxScores.gte(confThreshold);
        // ... NMS 적용

        // DetectedObjects 빌드
        List<String> detectedClasses = new ArrayList<>();
        List<Double> probabilities = new ArrayList<>();
        List<BoundingBox> boundingBoxes = new ArrayList<>();
        // ... 결과 조립

        return new DetectedObjects(detectedClasses, probabilities, boundingBoxes);
    }
}
```

---

## 6. YOLO 버전 역사

| 버전 | 연도 | 개발자 | 핵심 개선 |
|------|------|--------|----------|
| **YOLOv1** | 2015 | Joseph Redmon | 최초 단일 신경망 실시간 감지 (45fps) |
| **YOLOv2 (YOLO9000)** | 2016 | Redmon | 앵커 박스, 9000개 클래스, 배치 정규화 |
| **YOLOv3** | 2018 | Redmon | 다중 스케일 예측 (3레벨), Darknet-53, 608×608 |
| **YOLOv4** | 2020 | Bochkovskiy | CSP, SAM, PANnet, 모자이크 증강 |
| **YOLOv5** | 2020 | Ultralytics | PyTorch 네이티브, AutoAnchor, 실용성 높음 |
| **YOLOv6** | 2022 | Meituan | EfficientRep backbone, 산업용 최적화 |
| **YOLOv7** | 2022 | Wang et al. | E-ELAN, 모델 재파라미터화, SOTA 당시 |
| **YOLOv8** | 2023 | Ultralytics | 앵커 없음, Python API 개선, 분류/세그/포즈 통합 |
| **YOLOv9** | 2024 | Wang et al. | PGI (Programmable Gradient Info), GELAN |
| **YOLOv10** | 2024 | THU | NMS 없는 End-to-End 감지 |
| **YOLOv11** | 2024 | Ultralytics | C3K2 블록, 더 적은 파라미터로 높은 성능 |

### YOLOv8 모델 크기 비교 (COCO val2017)

| 모델 | 파라미터 | GFLOPs | mAP50-95 | 속도(A100) | 용도 |
|------|---------|--------|----------|-----------|------|
| **YOLOv8n** | 3.2M | 8.7 | 37.3 | 0.99ms | 엣지/모바일, 실시간 |
| **YOLOv8s** | 11.2M | 28.6 | 44.9 | 1.20ms | 경량 서버 |
| **YOLOv8m** | 25.9M | 78.9 | 50.2 | 1.83ms | 균형 (권장) |
| **YOLOv8l** | 43.7M | 165.2 | 52.9 | 2.39ms | 고정밀 |
| **YOLOv8x** | 68.2M | 257.8 | 53.9 | 3.53ms | 최고 정밀 |

### YOLOv9 모델 크기 비교 (COCO val2017)

| 모델 | 파라미터 | GFLOPs | mAP50-95 | 특징 |
|------|---------|--------|----------|------|
| **YOLOv9t** | 2.0M | 7.7 | 38.3 | Tiny, YOLOv8n보다 가볍고 정확 |
| **YOLOv9s** | 7.1M | 26.4 | 46.8 | YOLOv8s 대비 +1.9 mAP |
| **YOLOv9m** | 20.0M | 76.3 | 51.4 | |
| **YOLOv9c** | 25.3M | 102.1 | 53.0 | Compact |
| **YOLOv9e** | 57.3M | 189.0 | 55.6 | Extended, SOTA |

### YOLOv11 모델 크기 비교 (COCO val2017)

| 모델 | 파라미터 | GFLOPs | mAP50-95 | vs v8 동급 |
|------|---------|--------|----------|-----------|
| **YOLO11n** | 2.6M | 6.5 | 39.5 | v8n 대비 -19% 파라미터, +2.2 mAP |
| **YOLO11s** | 9.4M | 21.5 | 47.0 | v8s 대비 -16% 파라미터, +2.1 mAP |
| **YOLO11m** | 20.1M | 68.0 | 51.5 | |
| **YOLO11l** | 25.3M | 86.9 | 53.4 | |
| **YOLO11x** | 56.9M | 194.9 | 54.7 | v8x 대비 -17% 파라미터, +0.8 mAP |

### 버전별 선택 기준

```
[실시간 CCTV — 낮은 지연 우선]
  YOLOv8n / YOLO11n
  → Jetson Nano, Raspberry Pi 5에서 동작
  → Java(DJL): ~30ms/frame (CPU), ~5ms/frame (GPU)

[서버 스트리밍 분석 — 정확도 우선]
  YOLOv8m / YOLOv9c / YOLO11m
  → mAP 50+ 달성, 실용적 균형
  → Spring Boot + DJL + CUDA: ~10ms/frame

[연구·최고 정밀]
  YOLOv9e / YOLO11x
  → COCO mAP 55+
  → A100 GPU 필요

[ONNX 변환 권장 버전]
  → YOLOv8 또는 YOLO11 (Ultralytics 공식 지원, DJL 통합 사례 많음)
  → YOLOv5도 레거시 코드베이스에서 여전히 많이 사용
```

---

## 7. 배치 추론 최적화

```java
@Service
@Slf4j
public class BatchedDetectionService {

    private final ZooModel<Image, DetectedObjects> model;
    private final int batchSize = 4;

    // 카메라 여러 대의 프레임을 배치로 묶어 GPU 효율 최대화
    public List<List<Detection>> detectBatch(List<BufferedImage> frames) {
        List<List<Detection>> results = new ArrayList<>();

        // DJL 배치 추론 (모델이 배치 지원 시)
        try (NDManager manager = NDManager.newBaseManager()) {
            NDList batch = new NDList();
            for (BufferedImage frame : frames) {
                Image djlImage = ImageFactory.getInstance().fromImage(frame);
                NDArray tensor = djlImage.toNDArray(manager, Image.Flag.COLOR)
                    .transpose(2, 0, 1)
                    .div(255f);
                batch.add(tensor);
            }
            // 배치 차원으로 스택: [N, 3, 640, 640]
            NDArray batched = NDArrays.stack(batch);
            // ... 배치 추론
        }

        return results;
    }
}
```

---

## 8. Spring Boot 자동 설정

```yaml
# application.yml
djl:
  enabled: true
  default-engine: OnnxRuntime
  model-loading-timeout: 60s

cctv:
  ai:
    model-path: models/yolov8m.onnx
    confidence-threshold: 0.5
    nms-threshold: 0.45
    device: cuda:0          # GPU 0번 지정, "cpu"로 CPU 강제 지정
    input-size: 640
    classes-file: models/coco.names
```

```java
@ConfigurationProperties("cctv.ai")
@Data
public class CctvAiProperties {
    private String modelPath;
    private float confidenceThreshold = 0.5f;
    private float nmsThreshold = 0.45f;
    private String device = "auto";  // auto, cpu, cuda:0
    private int inputSize = 640;
    private String classesFile;
}
```

---

## 9. TensorRT — FP16 양자화로 최대 성능 뽑기

### TensorRT 개요

```
개발: NVIDIA
목적: NVIDIA GPU에서 딥러닝 추론 최대 최적화
핵심 기술:
  - Layer Fusion: 여러 레이어를 하나의 CUDA 커널로 합침
  - Kernel Auto-Tuning: GPU 모델에 맞는 최적 커널 선택
  - Precision Calibration: FP32 → FP16 → INT8 단계적 양자화

정밀도별 비교:
  FP32 (기본): 32비트 부동소수점, 기준
  FP16 (Half): 16비트, VRAM 절반, 처리 속도 2~3배↑, 정확도 미미한 손실
  INT8 (정수): 8비트, VRAM 1/4, 처리 속도 4~6배↑, 정확도 약간 손실 + 보정 데이터셋 필요
  BF16 (Brain Float): FP16과 비슷, 오버플로우 내성↑ (Ampere+ GPU)

제약:
  TensorRT 엔진 파일 (.engine/.trt)은 GPU 아키텍처 전용 (이식 불가)
  예: RTX 4090 빌드 → A100에서 사용 불가
  빌드 환경 = 실행 환경 동일해야 함
```

### YOLO → TensorRT 변환 (Python, 한 번만 실행)

```python
# 방법 1: Ultralytics CLI (가장 간단)
# pip install ultralytics tensorrt
from ultralytics import YOLO

model = YOLO("yolov8m.pt")

# FP32 ONNX 경유 TensorRT 변환
model.export(
    format="engine",        # TensorRT .engine 파일 생성
    half=True,              # FP16 활성화
    device=0,               # GPU 0 사용
    imgsz=640,
    batch=1,                # 배치 크기 (고정 또는 dynamic)
    workspace=4,            # TensorRT 작업 메모리 (GB)
)
# → yolov8m.engine 생성 (현재 GPU 전용)

# 방법 2: 동적 배치 크기 (여러 배치 크기 동시 지원)
model.export(
    format="engine",
    half=True,
    device=0,
    imgsz=640,
    batch=8,                # 최대 배치 크기
    dynamic=True,           # 동적 배치 (1~8 자유롭게 사용)
    simplify=True,          # ONNX 단순화 후 변환
)

# 방법 3: ONNX → TensorRT 직접 변환 (trtexec CLI)
# NVIDIA TensorRT SDK 설치 필요
# trtexec --onnx=yolov8m.onnx \
#         --saveEngine=yolov8m_fp16.engine \
#         --fp16 \
#         --workspace=4096 \
#         --minShapes=images:1x3x640x640 \
#         --optShapes=images:4x3x640x640 \
#         --maxShapes=images:8x3x640x640
```

### DJL TensorRT 엔진 설정

```xml
<!-- pom.xml — TensorRT 엔진 추가 -->
<dependency>
  <groupId>ai.djl.tensorrt</groupId>
  <artifactId>tensorrt</artifactId>
  <version>0.30.0</version>
</dependency>
```

```java
@Configuration
public class TensorRTModelConfig {

    @Bean
    @SneakyThrows
    public ZooModel<Image, DetectedObjects> yoloTrtModel() {
        YoloV8Translator translator = YoloV8Translator.builder()
            .optThreshold(0.5f)
            .optNmsThreshold(0.45f)
            .optSynsetArtifactName("coco.names")
            .build();

        Criteria<Image, DetectedObjects> criteria = Criteria.builder()
            .setTypes(Image.class, DetectedObjects.class)
            .optModelPath(Paths.get("models/yolov8m.engine"))  // TRT 엔진
            .optTranslator(translator)
            .optEngine("TensorRT")    // TensorRT 엔진 지정
            .optDevice(Device.gpu(0)) // GPU 0 필수
            .build();

        return ModelZoo.loadModel(criteria);
    }
}
```

### 성능 비교 (YOLOv8m, 1080p 이미지, RTX 3090)

| 정밀도 | 엔진 | 레이턴시(ms) | 처리량(FPS) | VRAM | mAP50-95 |
|--------|------|------------|------------|------|---------|
| FP32 | PyTorch | 18.5 | 54 | 2.6 GB | 50.2 |
| FP32 | ONNX Runtime | 14.2 | 70 | 2.5 GB | 50.2 |
| **FP16** | **TensorRT** | **5.8** | **172** | **1.4 GB** | **49.9** |
| INT8 | TensorRT | 3.2 | 312 | 0.8 GB | 48.1 |
| FP16 | TensorRT (배치 8) | 1.9 | 420 | 3.1 GB | 49.9 |

```
FP16 TensorRT 선택 이유:
  - PyTorch 대비 3배+ 빠름, VRAM 절반
  - mAP 손실: 50.2 → 49.9 (0.3p, 사람 눈에 무의미)
  - INT8 대비: 정확도 높음, 보정 데이터셋 불필요
  - CCTV 다채널(20대+) 실시간 분석에 FP16 TRT 사실상 필수
```

---

### INT8 보정 (선택 사항 — 최고 성능)

```python
# INT8: 보정 데이터셋으로 정밀도 손실 최소화
from ultralytics import YOLO

model = YOLO("yolov8m.pt")
model.export(
    format="engine",
    int8=True,
    data="coco128.yaml",   # 보정 데이터셋 (128장 이상 권장)
    device=0,
    imgsz=640,
    workspace=8,           # INT8 보정에 더 많은 메모리 필요
)
# 보정 없는 INT8 = 정확도 심각 저하
# 도메인 특화(공장/도로/매장) 이미지로 보정 시 mAP 손실 최소화
```

---

### 멀티 GPU / 다채널 배분

```java
@Service
@Slf4j
public class MultiGpuDetectionService {

    private final List<Predictor<Image, DetectedObjects>> predictors = new ArrayList<>();

    @PostConstruct
    public void init() throws Exception {
        // GPU가 2개라면 각 GPU에 별도 모델 로드
        int gpuCount = Device.getGpuCount();
        for (int i = 0; i < gpuCount; i++) {
            Criteria<Image, DetectedObjects> criteria = Criteria.builder()
                .setTypes(Image.class, DetectedObjects.class)
                .optModelPath(Paths.get("models/yolov8m.engine"))
                .optEngine("TensorRT")
                .optDevice(Device.gpu(i))   // GPU i번 할당
                .build();
            predictors.add(ModelZoo.loadModel(criteria).newPredictor());
        }
        log.info("Loaded TRT model on {} GPU(s)", gpuCount);
    }

    // 카메라 ID % GPU 수로 GPU 라운드로빈 배분
    public DetectedObjects detect(int cameraIndex, Image image) throws TranslateException {
        int gpuIdx = cameraIndex % predictors.size();
        return predictors.get(gpuIdx).predict(image);
    }
}
```

---

### TensorRT 적용 체크리스트

```
✅ NVIDIA 드라이버 525+ 설치
✅ CUDA 12.x 설치
✅ TensorRT 8.6+ 설치 (또는 Ultralytics가 자동 처리)
✅ 빌드 환경 = 배포 환경 동일한 GPU 아키텍처
✅ .engine 파일: 버전 관리에 포함 (용량 주의, LFS 사용)
✅ 워밍업: 첫 추론 3~5회는 느림 → 서버 시작 시 더미 추론 실행
✅ 스레드 안전: Predictor는 스레드 안전하지 않음 → 채널당 1개
```

```java
// 워밍업 — 서버 시작 시 실행
@PostConstruct
public void warmup() throws Exception {
    log.info("TensorRT warmup start...");
    BufferedImage dummy = new BufferedImage(640, 640, BufferedImage.TYPE_INT_RGB);
    Image dummyImage = ImageFactory.getInstance().fromImage(dummy);
    for (int i = 0; i < 5; i++) {
        predictor.predict(dummyImage);
    }
    log.info("TensorRT warmup done");
}
```

---

## 10. 관련
- [[YOLO-Finetuning]] — DJL로 배포할 YOLO 모델의 학습·파인튜닝 (데이터·증강·작은 객체 대응)
- [[Spring-CCTV]] — DJL을 JavaCV와 통합하는 CCTV 처리 아키텍처
- [[LLMOps]] — DJL의 LLM 추론 활용 (Whisper, 텍스트 분류)
- [[RAG]] — 비디오 메타데이터를 벡터 DB에 저장하는 파이프라인
- [[../infra/database/pgvector]] — 감지 이벤트 임베딩 저장
