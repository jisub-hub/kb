---
tags:
  - iot
  - protocol
  - mqtt
  - tcp
  - udp
  - lorawan
  - rest
created: 2026-06-16
---

# IoT 데이터 수집 경로 — Socket / TCP / UDP / MQTT / REST / LoRaWAN

> [!summary] 한 줄 요약
> IoT 디바이스에서 서버로 데이터를 전달하는 경로는 **네트워크 환경, 전력, 데이터 크기, 신뢰성 요건**에 따라 선택한다. 저전력 광역망(LoRaWAN)부터 실시간 스트림(TCP Socket)까지 각 프로토콜의 원리와 적합 시나리오를 정리한다.

---

## 1. 프로토콜 비교 요약

| 프로토콜 | 전송 계층 | 신뢰성 | 실시간성 | 전력 소비 | 적합 디바이스 |
|---------|---------|--------|---------|----------|-------------|
| **TCP Socket** | TCP | 보장 | 높음 | 중간 | 카메라, PLC, 라즈베리파이 |
| **UDP Socket** | UDP | 미보장 | 매우 높음 | 낮음 | 센서 스트림, 영상 전송 |
| **MQTT** | TCP | QoS 0/1/2 | 높음 | 낮음 | 소형 MCU, 센서 허브 ⭐ |
| **REST API** | HTTP/TCP | 보장 | 낮음 | 중간~높음 | 배치 업로드, 리소스 풍부 디바이스 |
| **LoRaWAN** | 무선 저전력 | QoS 미보장 | 낮음 (초~분) | 매우 낮음 | 배터리 센서, 농장·스마트시티 |
| **WebSocket** | TCP | 보장 | 높음 | 중간 | 웹 기반 제어 패널, 대시보드 |

---

## 2. TCP Socket (Raw TCP)

### 원리
```
[디바이스]   TCP connect()   [서버 Socket 서버]
              ──────────────►  accept()
              [데이터 스트림]  read()/write()
              ◄──────────────  양방향 지속 연결
```

- **지속 연결**: 한 번 연결 후 세션 유지 → 매 요청 핸드셰이크 없음
- **순서 보장**: TCP 레벨에서 순서·재전송 보장
- **프레이밍 직접 구현**: HTTP와 달리 헤더가 없으므로 **메시지 경계를 직접 정의**해야 함

### 주요 패턴

```
고정 길이: 매 패킷이 항상 N 바이트 → 단순
길이 접두사: [4바이트 길이][페이로드] → 가변 길이 대응
구분자: 0x0A(\n) 또는 0x00(null)로 메시지 구분 → 텍스트 프로토콜
```

### Java TCP 서버 (Spring)

```java
@Component
public class TcpSocketServer {

    @EventListener(ApplicationReadyEvent.class)
    public void start() {
        Executors.newSingleThreadExecutor().submit(() -> {
            try (var serverSocket = new ServerSocket(9000)) {
                log.info("TCP server listening on :9000");
                while (!Thread.currentThread().isInterrupted()) {
                    var client = serverSocket.accept();
                    Executors.newVirtualThreadPerTaskExecutor()
                        .submit(() -> handle(client));
                }
            }
        });
    }

    private void handle(Socket socket) {
        try (var in = new DataInputStream(socket.getInputStream())) {
            while (true) {
                int len = in.readInt();             // 4바이트 길이 접두사
                byte[] payload = in.readNBytes(len);
                processPayload(payload);
            }
        } catch (EOFException e) {
            log.info("Client disconnected: {}", socket.getRemoteSocketAddress());
        }
    }
}
```

### 적합 시나리오
- 산업용 PLC, SCADA 시스템 (독자 바이너리 프로토콜)
- 실시간 위치 추적 (GPS 트래커 → 서버)
- 고빈도 센서 (초당 수백 패킷)

---

## 3. UDP Socket

### 원리
```
[디바이스]   sendto()   [서버]
              ─────────► recvfrom()
              비연결형, 순서/신뢰성 없음, 오버헤드 최소
```

- **연결 없음**: 각 패킷이 독립적, 3-way handshake 없음
- **순서 보장 없음**: 패킷 도착 순서 역전 가능 → 시퀀스 번호 직접 구현
- **손실 허용**: 최신 값만 중요한 센서 데이터에 적합 (오래된 패킷 손실은 무의미)
- **최대 페이로드**: 이더넷 기준 ~1472 바이트 (MTU 1500 - IP/UDP 헤더)

### Java UDP 서버

```java
@Component
public class UdpSensorServer {

    @EventListener(ApplicationReadyEvent.class)
    public void start() {
        Executors.newSingleThreadExecutor().submit(() -> {
            try (var socket = new DatagramSocket(9001)) {
                var buf = new byte[1500];
                var packet = new DatagramPacket(buf, buf.length);
                while (true) {
                    socket.receive(packet);
                    var data = Arrays.copyOf(packet.getData(), packet.getLength());
                    var sensorId = packet.getAddress().getHostAddress();
                    processSensorData(sensorId, data);
                }
            }
        });
    }
}
```

### 적합 시나리오
- 영상 스트림 (RTP over UDP)
- 고빈도 센서 (온도·습도, 최신 값만 필요)
- 게임 컨트롤러, 드론 원격 조종 (낮은 지연 우선)
- LAN 환경 (패킷 손실 낮음)

---

## 4. MQTT (Message Queuing Telemetry Transport)

### 원리
```
[디바이스 - Publisher]
  CONNECT → Broker
  PUBLISH sensors/temp/device-1  payload: {"temp":23.5}

[서버 - Subscriber]
  CONNECT → Broker
  SUBSCRIBE sensors/temp/#       (+ 와일드카드)
  ← 메시지 수신
```

- **브로커 중심**: 디바이스와 서버가 직접 연결하지 않음 → 느슨한 결합
- **토픽 기반**: 계층적 토픽 구조 (`sensors/factory1/line2/temp`)
- **QoS 3단계**:
  - `QoS 0`: At most once (최대 1회, 손실 가능) — 센서 데이터
  - `QoS 1`: At least once (최소 1회, 중복 가능) — 명령 전달
  - `QoS 2`: Exactly once (정확히 1회, 오버헤드 최대) — 결제·제어

### 토픽 설계 패턴

```
[계층 구조]
factory/line1/machine3/temperature   ← 특정 기기
factory/+/machine3/temperature       ← + : 한 단계 와일드카드
factory/#                             ← # : 하위 전체

[기능 분리]
cmd/device-001/set     ← 명령 전달 (서버 → 디바이스)
state/device-001       ← 상태 보고 (디바이스 → 서버)
event/device-001/alert ← 이벤트 알림

[Last Will and Testament (LWT)]
디바이스 연결 시 LWT 등록:
  topic: status/device-001
  payload: {"online": false}
→ 디바이스 비정상 종료 시 브로커가 자동 발행 → 서버가 오프라인 감지
```

### Spring MQTT 연동

```groovy
implementation 'org.springframework.integration:spring-integration-mqtt'
```

```java
@Configuration
public class MqttConfig {

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        var factory = new DefaultMqttPahoClientFactory();
        var options = new MqttConnectOptions();
        options.setServerURIs(new String[]{"tcp://broker:1883"});
        options.setUserName("iot-user");
        options.setPassword("secret".toCharArray());
        options.setAutomaticReconnect(true);
        options.setKeepAliveInterval(30);
        options.setCleanSession(false);            // 재연결 시 미수신 메시지 수신
        factory.setConnectionOptions(options);
        return factory;
    }

    // Inbound — 수신
    @Bean
    public MessageChannel mqttInputChannel() { return new DirectChannel(); }

    @Bean
    public MessageProducerSupport mqttInbound(MqttPahoClientFactory factory) {
        var adapter = new MqttPahoMessageDrivenChannelAdapter(
            "spring-sub-client", factory,
            "sensors/#", "cmd/#");
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    public MessageHandler mqttMessageHandler() {
        return message -> {
            var topic   = (String) message.getHeaders().get(MqttHeaders.RECEIVED_TOPIC);
            var payload = (String) message.getPayload();
            log.info("MQTT recv topic={} payload={}", topic, payload);
        };
    }

    // Outbound — 발행
    @Bean
    public MqttPahoMessageHandler mqttOutbound(MqttPahoClientFactory factory) {
        var handler = new MqttPahoMessageHandler("spring-pub-client", factory);
        handler.setAsync(true);
        handler.setDefaultQos(1);
        return handler;
    }
}
```

### 브로커 선택

| 브로커 | 특징 | 적합 환경 |
|-------|------|----------|
| **Mosquitto** | 경량, 단일 프로세스 | 소규모, 에지 게이트웨이 |
| **EMQX** | 고성능, 클러스터, 대시보드 | 수백만 연결, 엔터프라이즈 |
| **HiveMQ** | 엔터프라이즈, K8s 오퍼레이터 | 대규모 상용 |
| **VerneMQ** | Erlang 기반, 분산 | 고가용성 |
| **Kafka** | 스트리밍 + 영속성 | 이벤트 파이프라인 연동 |

---

## 5. REST API

### 원리
```
[디바이스]
  POST /api/v1/sensors HTTP/1.1
  Content-Type: application/json
  Authorization: Bearer <token>
  {"deviceId":"dev-001","temp":23.5,"ts":1718500000}

[서버]
  200 OK / 201 Created
```

- **가장 단순**: 표준 HTTP → 방화벽 통과 쉬움, TLS 기본
- **오버헤드 높음**: 매 요청마다 TCP/TLS 핸드셰이크 (HTTP Keep-Alive로 완화)
- **배치 전송 권장**: 소형 디바이스는 N개 모아서 한 번 전송

### 배치 업로드 패턴

```java
// 디바이스 측 (의사코드)
List<SensorData> buffer = new ArrayList<>();
// 타이머: 30초마다 또는 버퍼가 50개 차면 전송
if (buffer.size() >= 50 || elapsed > 30s) {
    POST /api/v1/sensors/batch  body: buffer
    buffer.clear();
}
```

```java
// Spring 수신 엔드포인트
@RestController
@RequestMapping("/api/v1/sensors")
public class SensorController {

    @PostMapping("/batch")
    public ResponseEntity<Void> receiveBatch(
            @RequestBody @Valid List<SensorDataDto> batch,
            @RequestHeader("X-Device-Id") String deviceId) {
        sensorService.processBatch(deviceId, batch);
        return ResponseEntity.accepted().build();
    }
}
```

### 적합 시나리오
- 리소스 풍부 디바이스 (Raspberry Pi, 산업 PC)
- 인터넷 직접 통신 (3G/4G/Wi-Fi)
- 배치성 업로드 (5~30분 주기)
- 기존 HTTP 인프라 재사용

---

## 6. LoRaWAN

→ **[[LoRaWAN-ChirpStack]]** 전용 문서 참고.

```
LoRa 물리층: 확산 스펙트럼(CSS), 최대 15km, 초저전력 (AA 배터리 수년)
LoRaWAN 네트워크층: ADR, OTAA/ABP 인증, 클래스 A/B/C 디바이스

데이터 흐름:
  디바이스 → (무선) → LoRa Gateway → (IP) → Network Server (ChirpStack)
           → Application Server → MQTT publish → 처리 파이프라인
```

---

## 7. 프로토콜 선택 가이드

```
배터리 구동 + 넓은 면적 + 소량 데이터 (< 50바이트/패킷)
  → LoRaWAN + ChirpStack

소형 MCU (ESP32/Arduino) + 근거리 Wi-Fi/LTE
  → MQTT (QoS 1, 브로커: EMQX/Mosquitto)

고빈도 센서 + LAN 환경 + 최신 값만 필요
  → UDP Socket

PLC·산업 장비 + 바이너리 프로토콜 + 지속 연결
  → TCP Raw Socket

리소스 풍부 + 인터넷 + 배치 업로드
  → REST API (HTTPS)

실시간 제어 + 양방향 + 웹 대시보드
  → WebSocket (또는 MQTT over WebSocket)
```

---

## 8. 관련
- [[MQTT]] — MQTT 브로커, 토픽 설계, QoS 심화
- [[LoRaWAN-ChirpStack]] — LoRa 네트워크, ChirpStack 설치·운영
- [[Node-RED]] — 데이터 파싱·변환·라우팅
- [[IoT-Architecture]] — 전체 아키텍처: ChirpStack → MQTT → Node-RED → broker
