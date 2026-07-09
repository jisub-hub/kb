---
tags:
  - iot
  - node-red
  - flow
  - broker
  - mqtt
  - parser
created: 2026-06-16
---

# Node-RED — IoT 데이터 파싱·변환·라우팅

> [!summary] 한 줄 요약
> **Node-RED**는 브라우저 기반 비주얼 플로우 에디터로 IoT 데이터 파이프라인을 코딩 없이 또는 최소 JavaScript로 구성한다. MQTT 수신 → 파싱 → 변환 → 다양한 싱크(MQTT/HTTP/DB/알림) 연결을 담당하는 **소프트웨어 미들레이어**.

---

## 1. 역할과 위치

```
[데이터 소스]
  ChirpStack → MQTT broker (application/#)
  REST API POST
  TCP/UDP raw socket
  시리얼 포트
         │
     [Node-RED]         ← 이곳에서 파싱·변환·라우팅
  ┌────────────────┐
  │ MQTT in node   │  ← application/+/device/+/event/up 구독
  │ JSON parse     │  ← 문자열 → 객체
  │ function node  │  ← 필드 추출·변환·단위 변환
  │ switch node    │  ← 조건 분기 (디바이스 타입, 이상값)
  │ MQTT out node  │  ← 타겟 MQTT 브로커 발행
  │ HTTP request   │  ← 백엔드 REST API 호출
  │ TimescaleDB    │  ← 시계열 DB 저장
  │ Telegram/Slack │  ← 알림
  └────────────────┘
         │
  [타겟 시스템]
```

---

## 2. 설치

```bash
# Docker
docker run -d \
  --name node-red \
  -p 1880:1880 \
  -v node-red-data:/data \
  nodered/node-red:latest

# K8s
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-red
spec:
  replicas: 1
  selector:
    matchLabels: { app: node-red }
  template:
    metadata:
      labels: { app: node-red }
    spec:
      containers:
        - name: node-red
          image: nodered/node-red:latest
          ports: [{ containerPort: 1880 }]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: node-red-pvc
EOF

# UI 접근
kubectl port-forward svc/node-red 1880:1880
# http://localhost:1880
```

### 커뮤니티 노드 설치

```bash
# Node-RED UI → 메뉴 → Manage palette → Install
# 또는 /data 디렉토리에서:
npm install node-red-contrib-influxdb
npm install node-red-contrib-postgresql
npm install node-red-dashboard     # UI 대시보드
npm install node-red-node-serialport
```

---

## 3. 플로우 구성 — ChirpStack MQTT 수신

```json
// flows.json — 기본 파이프라인
[
  {
    "id": "mqtt-in-chirpstack",
    "type": "mqtt in",
    "broker": "mqtt-broker-config",
    "topic": "application/+/device/+/event/up",
    "qos": "1",
    "wires": [["json-parse"]]
  },
  {
    "id": "json-parse",
    "type": "json",
    "wires": [["field-extractor"]]
  },
  {
    "id": "field-extractor",
    "type": "function",
    "func": "
// ChirpStack uplink 페이로드에서 필요한 필드 추출
var obj = msg.payload;
var decoded = obj.object;          // 코덱이 디코딩한 결과

if (!decoded) {
    // 코덱 없는 경우 base64 raw bytes 직접 디코딩
    var bytes = Buffer.from(obj.data, 'base64');
    decoded = {
        temperature: ((bytes[0] << 8) | bytes[1]) / 100.0,
        humidity:    bytes[2]
    };
}

msg.payload = {
    deviceEui:   obj.deviceInfo.devEui,
    deviceName:  obj.deviceInfo.deviceName,
    location:    obj.deviceInfo.tags.location,
    temperature: decoded.temperature,
    humidity:    decoded.humidity,
    battery:     decoded.battery,
    rssi:        obj.rxInfo[0]?.rssi,
    snr:         obj.rxInfo[0]?.snr,
    timestamp:   obj.time || new Date().toISOString()
};
return msg;
",
    "wires": [["threshold-switch"]]
  },
  {
    "id": "threshold-switch",
    "type": "switch",
    "property": "payload.temperature",
    "rules": [
      { "t": "gt", "v": "35",  "vt": "num" },  // 35도 초과 → 경보
      { "t": "else" }                             // 정상
    ],
    "wires": [["alert-node"], ["target-mqtt-out"]]
  }
]
```

---

## 4. 주요 노드 패턴

### 4.1 Function 노드 — JavaScript 변환

```javascript
// 단위 변환 + 데이터 보강
var payload = msg.payload;

// 섭씨 → 화씨
payload.tempF = (payload.temperature * 9/5) + 32;

// 배터리 레벨 분류
payload.batteryStatus = payload.battery > 50 ? "OK"
    : payload.battery > 20 ? "LOW" : "CRITICAL";

// 타임스탬프 포맷 통일
payload.ts = new Date(payload.timestamp).getTime();  // Unix ms

// 컨텍스트 활용 (노드 간 상태 공유)
var flow = flow.get("deviceCache") || {};
flow[payload.deviceEui] = { lastSeen: Date.now(), ...payload };
flow.set("deviceCache", flow);

msg.payload = payload;
return msg;
```

### 4.2 Switch 노드 — 다중 조건 라우팅

```
조건 1: payload.deviceInfo.tags.type == "soil"   → 토양 처리 파이프라인
조건 2: payload.deviceInfo.tags.type == "water"  → 수위 처리 파이프라인
조건 3: payload.rssi < -110                       → 신호 약한 게이트웨이 알림
else                                               → 기본 파이프라인
```

### 4.3 Delay 노드 — 레이트 제한

```
디바이스가 너무 빠르게 패킷 전송 시 다운스트림 보호:
  Rate Limit: 1 msg/second per topic
  또는 Delay: 100ms 고정 지연
```

### 4.4 Change 노드 — 메시지 변환

```
msg.topic  → "iot/processed/{deviceEui}"  (표현식)
msg.payload.source → "chirpstack"          (고정값 추가)
delete msg.payload.rxInfo                  (불필요 필드 제거)
```

---

## 5. 타겟 MQTT 브로커로 발행

```json
// MQTT out 노드 — 내부 MQTT 브로커로 정규화된 데이터 발행
{
  "type": "mqtt out",
  "broker": "target-broker-config",
  "topic": "iot/processed/sensor",    // 고정 또는 msg.topic
  "qos": "1",
  "retain": false
}

// 토픽 동적 설정 (function 노드에서)
msg.topic = `iot/processed/${msg.payload.location}/${msg.payload.deviceEui}`;
```

---

## 6. 다양한 싱크 연결

### TimescaleDB / PostgreSQL 저장

```javascript
// PostgreSQL 노드 설정
// Query:
INSERT INTO sensor_readings (time, device_eui, temperature, humidity, battery, rssi)
VALUES ($1, $2, $3, $4, $5, $6)
ON CONFLICT DO NOTHING;

// msg.payload 매핑:
msg.payload = [
    msg.payload.timestamp,
    msg.payload.deviceEui,
    msg.payload.temperature,
    msg.payload.humidity,
    msg.payload.battery,
    msg.payload.rssi
];
```

### REST API 백엔드 호출

```json
{
  "type": "http request",
  "method": "POST",
  "url": "http://backend:8080/api/v1/iot/ingest",
  "headers": {
    "Content-Type": "application/json",
    "X-Api-Key": "{{env.IOT_API_KEY}}"
  }
}
```

### Telegram/Slack 알림 (이상값)

```javascript
// Function 노드 — Telegram 메시지 구성
msg.payload = {
    chatId: process.env.TELEGRAM_CHAT_ID,
    type: "message",
    content: `🌡️ 온도 이상 감지!\n` +
             `디바이스: ${msg.payload.deviceName}\n` +
             `온도: ${msg.payload.temperature}°C\n` +
             `위치: ${msg.payload.location}\n` +
             `시각: ${new Date().toLocaleString('ko-KR')}`
};
```

---

## 7. 환경변수 & 보안

```bash
# Node-RED settings.js 에서 환경변수 사용
# /data/settings.js
module.exports = {
    functionGlobalContext: {
        env: process.env    // function 노드에서 global.get("env") 접근
    }
}
```

```yaml
# Docker 환경변수 주입
services:
  node-red:
    image: nodered/node-red:latest
    environment:
      - MQTT_BROKER_URL=tcp://emqx:1883
      - MQTT_USERNAME=nodered
      - MQTT_PASSWORD_FILE=/run/secrets/mqtt_pass   # Docker secret
      - IOT_API_KEY_FILE=/run/secrets/api_key
      - TELEGRAM_CHAT_ID=-100123456789
    secrets:
      - mqtt_pass
      - api_key
```

---

## 9. 통합 파이프라인 — 다중 소스 → 단일 파싱 → Target MQTT

LoRaWAN은 ChirpStack이 전담하고, Node-RED는 ChirpStack MQTT를 포함한 모든 소스를 한 곳에서 수신·파싱·발행한다.

```
[소스별 입력 경로]
  ┌─── TCP 소켓 (직접 연결 디바이스)
  │    node: tcp in (포트 9001)
  │
  ├─── UDP 소켓 (경량 센서, 브로드캐스트)
  │    node: udp in (포트 9002)
  │
  ├─── HTTP POST (REST 디바이스, 모바일)
  │    node: http in (POST /sensor/data)
  │
  ├─── ChirpStack MQTT (LoRaWAN 전용)
  │    node: mqtt in
  │    topic: application/+/device/+/event/up
  │    broker: raw-mosquitto (ChirpStack 내부)
  │
  └─── 기타 MQTT (직접 Pub 디바이스)
       node: mqtt in
       topic: raw/device/+

              │ (모든 소스가 여기로)
              ▼
       [normalizer function]       ← 단일 정규화 함수 노드
        소스 구분 → 통일된 JSON 구조
              │
              ▼
       [validator / filter]        ← 이상값, 누락 필드 검증
              │
         ┌────┴────┐
         ▼         ▼
    [alert 분기]  [target mqtt out]
    임계값 초과    topic: iot/processed/{location}/{deviceId}
    → Telegram    broker: EMQX (소프트웨어 브로커)
```

### flows.json — 완전한 통합 파이프라인

```json
[
  // ── 1. TCP 수신 ────────────────────────────────────────────────
  {
    "id": "tcp-source",
    "type": "tcp in",
    "server": "",
    "port": "9001",
    "datamode": "stream",
    "datatype": "utf8",
    "wires": [["source-router"]]
  },

  // ── 2. UDP 수신 ────────────────────────────────────────────────
  {
    "id": "udp-source",
    "type": "udp in",
    "port": "9002",
    "datatype": "utf8",
    "wires": [["source-router"]]
  },

  // ── 3. HTTP POST 수신 ─────────────────────────────────────────
  {
    "id": "http-source",
    "type": "http in",
    "url": "/sensor/data",
    "method": "post",
    "wires": [["http-ack", "source-router"]]
  },
  {
    "id": "http-ack",
    "type": "http response",
    "statusCode": "200"
  },

  // ── 4. ChirpStack MQTT 수신 (LoRaWAN) ────────────────────────
  {
    "id": "chirpstack-mqtt-source",
    "type": "mqtt in",
    "broker": "raw-mosquitto-broker",
    "topic": "application/+/device/+/event/up",
    "qos": "1",
    "wires": [["source-router"]]
  },

  // ── 5. 기타 MQTT 직접 수신 ────────────────────────────────────
  {
    "id": "raw-mqtt-source",
    "type": "mqtt in",
    "broker": "raw-mosquitto-broker",
    "topic": "raw/device/+",
    "qos": "0",
    "wires": [["source-router"]]
  },

  // ── 6. 소스 라우터 — 소스별 정규화 분기 ─────────────────────
  {
    "id": "source-router",
    "type": "function",
    "func": "
// 소스 구분 후 정규화 함수로 라우팅
var topic = msg.topic || '';
var inputType = msg._inputType || '';

if (topic.startsWith('application/') && topic.includes('/event/up')) {
    msg._source = 'chirpstack';
} else if (topic.startsWith('raw/device/')) {
    msg._source = 'raw-mqtt';
} else if (msg.req) {
    msg._source = 'http';
    msg.payload = msg.payload;  // body already parsed
} else if (inputType === 'tcp') {
    msg._source = 'tcp';
} else {
    msg._source = 'udp';
}
return msg;
",
    "wires": [["normalizer"]]
  },

  // ── 7. 정규화 — 모든 소스를 통일된 JSON으로 ──────────────────
  {
    "id": "normalizer",
    "type": "function",
    "func": "
var source = msg._source;
var raw = msg.payload;
var normalized;

try {
    switch (source) {

      case 'chirpstack': {
        // ChirpStack uplink: msg.payload는 이미 JSON 객체
        var obj = (typeof raw === 'string') ? JSON.parse(raw) : raw;
        var decoded = obj.object || {};
        normalized = {
          deviceId:    obj.deviceInfo.devEui,
          deviceName:  obj.deviceInfo.deviceName,
          location:    (obj.deviceInfo.tags || {}).location || 'unknown',
          protocol:    'lorawan',
          temperature: decoded.temperature,
          humidity:    decoded.humidity,
          battery:     decoded.battery,
          rssi:        (obj.rxInfo && obj.rxInfo[0]) ? obj.rxInfo[0].rssi : null,
          snr:         (obj.rxInfo && obj.rxInfo[0]) ? obj.rxInfo[0].snr : null,
          ts:          obj.time || new Date().toISOString()
        };
        break;
      }

      case 'http':
      case 'raw-mqtt': {
        // 공통 JSON 포맷 가정: {deviceId, location, temperature, ...}
        var body = (typeof raw === 'string') ? JSON.parse(raw) : raw;
        normalized = {
          deviceId:    body.deviceId || body.device_id,
          deviceName:  body.deviceName || body.deviceId,
          location:    body.location || 'unknown',
          protocol:    source === 'http' ? 'http' : 'mqtt',
          temperature: body.temperature || body.temp,
          humidity:    body.humidity,
          battery:     body.battery,
          rssi:        null,
          snr:         null,
          ts:          body.timestamp || new Date().toISOString()
        };
        break;
      }

      case 'tcp':
      case 'udp': {
        // 예시: 길이 접두사 없는 CSV 또는 간단 JSON
        var text = (typeof raw === 'object') ? raw.toString() : raw;
        // 예: 'DEV001,field-a,23.5,60,87'
        if (text.trim().startsWith('{')) {
          var parsed = JSON.parse(text);
          normalized = { ...parsed, protocol: source };
        } else {
          var parts = text.trim().split(',');
          normalized = {
            deviceId:    parts[0],
            location:    parts[1],
            protocol:    source,
            temperature: parseFloat(parts[2]),
            humidity:    parseFloat(parts[3]),
            battery:     parseFloat(parts[4]),
            ts:          new Date().toISOString()
          };
        }
        break;
      }

      default:
        node.warn('Unknown source: ' + source);
        return null;
    }

    // 필수 필드 검증
    if (!normalized.deviceId) {
        node.warn('Missing deviceId from ' + source);
        return null;
    }

    msg.payload = normalized;
    msg.topic = 'iot/processed/' + (normalized.location) + '/' + normalized.deviceId;
    return msg;

} catch(e) {
    node.error('Normalize error [' + source + ']: ' + e.message);
    return null;
}
",
    "wires": [["threshold-check"]]
  },

  // ── 8. 임계값 검사 ────────────────────────────────────────────
  {
    "id": "threshold-check",
    "type": "switch",
    "property": "payload.temperature",
    "rules": [
      { "t": "gt", "v": "35", "vt": "num" },
      { "t": "else" }
    ],
    "wires": [["alert-composer"], ["target-mqtt-publish"]]
  },

  // ── 9. 알림 (임계값 초과) ─────────────────────────────────────
  {
    "id": "alert-composer",
    "type": "function",
    "func": "
var p = msg.payload;
msg.payload = {
    chatId: env.get('TELEGRAM_CHAT_ID'),
    type: 'message',
    content: '[경보] ' + p.deviceName + '\\n' +
             '온도: ' + p.temperature + '°C\\n' +
             '위치: ' + p.location + '\\n' +
             '프로토콜: ' + p.protocol
};
return msg;
",
    "wires": [["telegram-out"], ["target-mqtt-publish"]]
  },

  // ── 10. Target MQTT 발행 (EMQX) ───────────────────────────────
  {
    "id": "target-mqtt-publish",
    "type": "mqtt out",
    "broker": "emqx-broker",
    "topic": "",          // msg.topic 사용
    "qos": "1",
    "retain": false,
    "wires": []
  }
]
```

### 브로커 설정 노드 (`flows.json` 추가)

```json
[
  {
    "id": "raw-mosquitto-broker",
    "type": "mqtt-broker",
    "name": "ChirpStack Raw MQTT",
    "broker": "mosquitto-raw",
    "port": "1883",
    "clientid": "nodered-chirpstack-consumer",
    "usetls": false,
    "cleansession": false,      // QoS 1 큐 보존 (Node-RED 재시작 시 미처리 메시지 재수신)
    "keepalive": "60"
  },
  {
    "id": "emqx-broker",
    "type": "mqtt-broker",
    "name": "EMQX Target Broker",
    "broker": "emqx",
    "port": "1883",
    "clientid": "nodered-publisher",
    "credentials": {
      "user": "nodered",
      "password": ""            // 실제 운용 시 환경변수로 주입
    },
    "keepalive": "60",
    "usetls": false
  }
]
```

### 환경변수로 자격증명 주입

```yaml
# docker-compose.yml
services:
  node-red:
    image: nodered/node-red:latest
    environment:
      - EMQX_USER=nodered
      - EMQX_PASS=${EMQX_NODERED_PASS}   # .env에서 주입
      - TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
```

```javascript
// function 노드에서 환경변수 참조
var mqttUser = env.get('EMQX_USER');
var chatId   = env.get('TELEGRAM_CHAT_ID');
```

---

## 10. 관련
- [[MQTT]] — 브로커 설정, 토픽 설계, QoS
- [[LoRaWAN-ChirpStack]] — 업스트림 데이터 소스
- [[IoT-Architecture]] — Node-RED가 포함된 전체 아키텍처
- [[../../infra/database/TimescaleDB]] — 시계열 저장소
