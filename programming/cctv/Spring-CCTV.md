---
tags:
  - cctv
  - spring
  - javacv
  - ffmpeg
  - djl
  - video-processing
created: 2026-06-16
---

# Spring 기반 CCTV 처리 — JavaCV vs FFmpeg 멀티프로세스

> [!summary] 한 줄 요약
> **JavaCV(인프로세스)**는 DJL AI 추론과 통합 가능하나 JNI 스레드 한계로 다채널 확장 시 `CPU코어수 × 포트수` 분산이 필요. **FFmpeg 서브프로세스**는 MediaMTX 단일 포트로 다채널 처리 효율이 높지만 JVM 내 DJL 사용 불가 — AI가 필요하면 별도 추론 서비스로 분리해야 한다.

---

## 1. 두 방식의 핵심 차이

```
[JavaCV — In-Process]
  Spring Boot JVM ─────────────────────────────────────────────────────
  │  FrameGrabber ──decode──▶ Mat(OpenCV) ──▶ DJL Predictor(AI)      │
  │  Thread-1 → cam1         인프레임 처리      YOLOv8 객체 감지        │
  │  Thread-2 → cam2                            DJL → Java API         │
  │  Thread-N → camN                                                   │
  └────────────────────────────────────────────────────────────────────

  장점: DJL로 JVM 내 AI 추론, 단일 배포
  한계: 스레드 1개 = 카메라 1개 (JNI blocking I/O)
        → 100채널 = 100 JNI 스레드 상시 점유

[FFmpeg 서브프로세스 — External Process]
  Spring Boot JVM                         FFmpeg 프로세스들
  │  ProcessBuilder ──spawn──▶ ffmpeg -i rtsp://cam1 → MediaMTX
  │  ProcessBuilder ──spawn──▶ ffmpeg -i rtsp://cam2 → MediaMTX
  │  ProcessBuilder ──spawn──▶ ffmpeg -i rtsp://camN → MediaMTX
  │                                MediaMTX :8554 (단일 포트)
  │  RTSP/HLS 클라이언트 ◀── MediaMTX 재배포
  └─────────────────────────────────────────────────

  장점: 채널당 OS 프로세스 → 확장성 우수, 단일 포트
  한계: JVM 내 DJL 미사용, AI 필요 시 별도 서비스 필요
```

---

## 2. JavaCV — In-Process 처리

### 의존성

```xml
<dependency>
  <groupId>org.bytedeco</groupId>
  <artifactId>javacv-platform</artifactId>
  <version>1.5.10</version>
</dependency>
<!-- DJL (Deep Java Library) AI 추론 -->
<dependency>
  <groupId>ai.djl.pytorch</groupId>
  <artifactId>pytorch-engine</artifactId>
  <version>0.29.0</version>
</dependency>
<dependency>
  <groupId>ai.djl.pytorch</groupId>
  <artifactId>pytorch-native-auto</artifactId>
  <version>2.3.1</version>
</dependency>
```

### RTSP 프레임 수집

```java
@Component
@Slf4j
public class RtspFrameGrabber {

    // JNI 특성상 스레드 1개 = 카메라 1개 고정
    // Virtual Thread (JDK 21) 사용해도 JNI blocking I/O는 캐리어 스레드 고정됨 ← 핵심 한계
    public void startCapture(String rtspUrl, FrameConsumer consumer) {
        Thread.ofPlatform().daemon().start(() -> {
            try (FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(rtspUrl)) {
                grabber.setOption("rtsp_transport", "tcp");
                grabber.setOption("fflags", "nobuffer");
                grabber.setOption("flags", "low_delay");
                grabber.setVideoCodec(avcodec.AV_CODEC_ID_H264);
                grabber.start();

                OpenCVFrameConverter.ToMat converter = new OpenCVFrameConverter.ToMat();

                while (!Thread.currentThread().isInterrupted()) {
                    Frame frame = grabber.grabImage();
                    if (frame != null && frame.image != null) {
                        Mat mat = converter.convert(frame);
                        consumer.process(mat);  // DJL 추론 등
                        mat.release();
                    }
                }
            } catch (Exception e) {
                log.error("Stream capture failed: {}", rtspUrl, e);
            }
        });
    }
}
```

### DJL 객체 감지 통합

```java
@Service
@Slf4j
public class VideoAnalyticsService {

    private ZooModel<Image, DetectedObjects> model;
    private Predictor<Image, DetectedObjects> predictor;

    @PostConstruct
    public void loadModel() throws Exception {
        // YOLOv8 또는 SSD 모델 로드
        Criteria<Image, DetectedObjects> criteria = Criteria.builder()
            .optApplication(Application.CV.OBJECT_DETECTION)
            .setTypes(Image.class, DetectedObjects.class)
            .optFilter("backbone", "mobilenet1.0")
            .optProgress(new ProgressBar())
            .build();
        model = ModelZoo.loadModel(criteria);
        predictor = model.newPredictor();
    }

    public List<DetectionResult> analyze(Mat frame) {
        // Mat → DJL Image 변환
        BufferedImage buffered = matToBufferedImage(frame);
        Image image = ImageFactory.getInstance().fromImage(buffered);

        try {
            DetectedObjects objects = predictor.predict(image);
            return objects.items().stream()
                .filter(item -> item.getProbability() > 0.7)
                .map(item -> new DetectionResult(
                    item.getClassName(),
                    item.getProbability(),
                    item.getBoundingBox()
                ))
                .toList();
        } catch (TranslateException e) {
            log.warn("Detection failed", e);
            return List.of();
        }
    }

    // Mat → BufferedImage 변환 (JNI 경계 처리)
    private BufferedImage matToBufferedImage(Mat mat) {
        MatOfByte buf = new MatOfByte();
        Imgcodecs.imencode(".jpg", mat, buf);
        return ImageIO.read(new ByteArrayInputStream(buf.toArray()));
    }
}
```

### 다채널 확장 한계와 해결책

```
[문제]
  카메라 100대 × 30fps = 초당 3,000 프레임 처리
  JavaCV는 채널당 1 JNI 스레드 고정
  → 100 스레드 상시 Block (Platform Thread, JNI 해제 불가)
  → JVM 힙 + 네이티브 메모리 동시 증가

[CPU × 포트 분산 전략]
  서버 1 (포트 8080): cam01~cam25  (CPU 코어 25개 할당)
  서버 2 (포트 8080): cam26~cam50
  서버 3 (포트 8080): cam51~cam75
  서버 4 (포트 8080): cam76~cam100

  → 로드밸런서(Nginx/L4)가 카메라 ID 기반으로 라우팅
  → K8s: replicas=4, 각 Pod에 카메라 25개 배정

[설정 예시]
  application.yml:
    cctv:
      cameras:
        - id: cam01
          rtsp: rtsp://192.168.1.101/stream1
        # ... 최대 25개/인스턴스
      maxChannelsPerInstance: 25
```

---

## 3. FFmpeg 서브프로세스 — External Process 방식

### ProcessBuilder 기반 스트림 관리

```java
@Service
@Slf4j
public class FfmpegStreamManager {

    private final Map<String, Process> processes = new ConcurrentHashMap<>();

    /**
     * RTSP 카메라를 MediaMTX로 릴레이하는 FFmpeg 프로세스 시작.
     * MediaMTX가 단일 포트(8554)에서 RTSP/HLS/WebRTC 동시 제공.
     */
    public void startRelay(String cameraId, String rtspUrl) {
        String targetPath = "rtsp://mediamtx:8554/" + cameraId;

        ProcessBuilder pb = new ProcessBuilder(
            "ffmpeg",
            "-rtsp_transport", "tcp",
            "-reconnect", "1",
            "-reconnect_at_eof", "1",
            "-reconnect_streamed", "1",
            "-reconnect_delay_max", "30",
            "-i", rtspUrl,
            "-c", "copy",           // 트랜스코딩 없음 — CPU 부하 최소
            "-f", "rtsp",
            targetPath
        );
        pb.redirectErrorStream(true);

        try {
            Process process = pb.start();
            processes.put(cameraId, process);
            monitorProcess(cameraId, process);
            log.info("Started FFmpeg relay: {} -> {}", cameraId, targetPath);
        } catch (IOException e) {
            throw new RuntimeException("Failed to start FFmpeg for " + cameraId, e);
        }
    }

    private void monitorProcess(String cameraId, Process process) {
        // 프로세스 로그 수집 (별도 스레드, 경량)
        Thread.ofVirtual().start(() -> {
            try (var reader = new BufferedReader(
                    new InputStreamReader(process.getInputStream()))) {
                String line;
                while ((line = reader.readLine()) != null) {
                    if (line.contains("error") || line.contains("Error")) {
                        log.warn("[{}] FFmpeg: {}", cameraId, line);
                    }
                }
            } catch (IOException ignored) {}

            // 프로세스 종료 감지 → 자동 재시작
            int exitCode = process.exitValue();
            log.warn("[{}] FFmpeg exited with code {}, restarting...", cameraId, exitCode);
            processes.remove(cameraId);
            // 재시작 로직 (지수 백오프 포함)
        });
    }

    public void stopRelay(String cameraId) {
        Process p = processes.remove(cameraId);
        if (p != null) {
            p.destroy();
        }
    }

    @PreDestroy
    public void stopAll() {
        processes.values().forEach(Process::destroy);
    }
}
```

### MediaMTX API 연동 — 스트림 상태 조회

```java
@Service
@RequiredArgsConstructor
public class MediaMtxClient {

    private final RestTemplate restTemplate;

    @Value("${mediamtx.api-url:http://mediamtx:9997}")
    private String apiUrl;

    public List<StreamInfo> listStreams() {
        var response = restTemplate.getForObject(
            apiUrl + "/v3/paths/list", PathListResponse.class);
        return response.getItems().stream()
            .map(item -> new StreamInfo(item.getName(), item.isReady(),
                         item.getReaders().size()))
            .toList();
    }

    public boolean isStreamReady(String cameraId) {
        try {
            var info = restTemplate.getForObject(
                apiUrl + "/v3/paths/get/" + cameraId, PathInfo.class);
            return info != null && info.isReady();
        } catch (Exception e) {
            return false;
        }
    }
}
```

### FFmpeg + 외부 AI 서비스 분리

```
[아키텍처]
  카메라 → FFmpeg(스트림 복사) → MediaMTX ──┬─▶ VMS/클라이언트
                                             │
                                FFmpeg(분기)  └─▶ AI 추론 서비스
                                  │               (Python FastAPI
                                  │                + YOLOv8/DJL)
                                  ▼                     │
                               RTSP 루프백 ──────────────┘
                               또는 파이프

[구현 예시 — FFmpeg → HTTP 스냅샷 → AI 서비스]
```

```java
@Component
@RequiredArgsConstructor
public class AiAnalysisPipeline {

    private final WebClient aiServiceClient;
    private final FfmpegStreamManager streamManager;

    @Scheduled(fixedDelay = 1000) // 1초마다 스냅샷 캡처
    public void captureAndAnalyze() {
        for (String cameraId : activeCameras) {
            // FFmpeg로 RTSP에서 단일 프레임 추출
            byte[] jpeg = captureFrame("rtsp://mediamtx:8554/" + cameraId);

            // 비동기 AI 분석 (별도 서비스)
            aiServiceClient.post()
                .uri("/analyze")
                .bodyValue(Map.of("cameraId", cameraId, "image", jpeg))
                .retrieve()
                .bodyToMono(AnalysisResult.class)
                .subscribe(result -> handleDetection(cameraId, result));
        }
    }

    private byte[] captureFrame(String rtspUrl) throws Exception {
        Process p = new ProcessBuilder(
            "ffmpeg", "-rtsp_transport", "tcp",
            "-i", rtspUrl,
            "-vframes", "1", "-f", "image2", "-vcodec", "mjpeg", "pipe:1"
        ).start();
        return p.getInputStream().readAllBytes();
    }
}
```

---

## 4. 방식 비교 요약

| 기준 | JavaCV (In-Process) | FFmpeg 서브프로세스 |
|------|--------------------|--------------------|
| **DJL AI 추론** | ✅ 직접 통합 | ❌ 별도 서비스 필요 |
| **다채널 확장** | CPU코어 × 포트수로 분산 | 단일 MediaMTX 포트 |
| **스레드 모델** | 채널당 JNI Platform Thread | 채널당 OS Process |
| **메모리** | JNI + 힙 동시 증가 | 프로세스 분리 (안전) |
| **배포 단순성** | 단일 JAR | JAR + FFmpeg + MediaMTX |
| **장애 격리** | 하나 죽으면 JVM 위험 | 프로세스 격리 |
| **트랜스코딩 유연성** | OpenCV 필터 | FFmpeg 필터그래프 |
| **프로토콜 제공** | Spring MVC (수동) | MediaMTX (RTSP/HLS/WebRTC 자동) |

### 선택 기준

```
[JavaCV 선택]
  → 실시간 AI 분석이 핵심 (객체 감지, 이상 행동 탐지)
  → DJL + GPU 가속 추론이 필요
  → 채널 수 50개 미만
  → 단일 서비스 배포 선호

[FFmpeg 서브프로세스 선택]
  → 스트리밍·재배포가 주 목적 (AI는 옵션)
  → 채널 수 100개+ 대규모
  → 장애 격리 중요 (카메라 하나 문제가 서버 전체에 영향 없도록)
  → MediaMTX의 멀티프로토콜(RTSP/HLS/WebRTC) 자동 제공 활용

[하이브리드 선택]
  → 스트리밍: FFmpeg + MediaMTX (단일 포트, 고효율)
  → AI 분석: 별도 Python 서비스 (YOLOv8 + CUDA)
  → 이벤트 통합: Spring이 AI 결과 수신 → DB 저장 → 알림
```

---

## 5. 운영 고려사항

```
[JavaCV 운영]
  JVM 힙: -Xmx8g (채널당 ~80MB 네이티브 버퍼 별도)
  GC: ZGC 또는 Shenandoah (STW 최소화 — 프레임 드롭 방지)
  스레드 모니터링: JFR + async-profiler로 JNI 스레드 병목 확인
  네이티브 메모리 누수: mat.release() 누락 시 OOM 발생 → try-with-resources 필수

[FFmpeg 서브프로세스 운영]
  좀비 프로세스 방지: @PreDestroy + Runtime.addShutdownHook
  자동 재시작: Supervisor 패턴 (지수 백오프: 5s, 10s, 30s, 60s)
  프로세스 수 제한: ulimit -u 확인 (기본 4096, 필요 시 상향)
  로그 수집: stderr 스레드로 반드시 소비 (파이프 버퍼 블록 방지)
  K8s: MediaMTX는 sidecar 컨테이너 또는 별도 Deployment
```

---

## 6. 관련
- [[MediaMTX]] — FFmpeg 방식에서 사용하는 미디어 서버
- [[FFmpeg]] — 서브프로세스로 실행하는 트랜스코딩·스트리밍 도구
- [[Streaming-Protocols]] — RTSP, ONVIF, PTZ 프로토콜
- [[Video-Fundamentals]] — 코덱, 색상 포맷 기초
- [[../../infra/k8s/Security-Context]] — K8s 배포 시 보안 컨텍스트
