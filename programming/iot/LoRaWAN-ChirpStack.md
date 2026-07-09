---
tags:
  - iot
  - lorawan
  - chirpstack
  - lora
  - lpwan
created: 2026-06-16
---

# LoRaWAN & ChirpStack

> [!summary] 한 줄 요약
> **LoRa**는 2~15km 도달하는 저전력 무선 물리층. **LoRaWAN**은 그 위의 MAC 프로토콜. **ChirpStack**은 오픈소스 LoRaWAN Network Server로 게이트웨이에서 수신한 패킷을 MQTT로 Application 서버에 전달한다.

---

## 1. LoRa / LoRaWAN 기술 스택

```
[애플리케이션]   JSON 페이로드 처리
      │
[LoRaWAN]       MAC 계층 — 인증(OTAA/ABP), ADR, 클래스 A/B/C, 암호화
      │
[LoRa 물리층]   CSS(Chirp Spread Spectrum) 변조
                주파수: 한국 920~923MHz (AS923), 유럽 868MHz, 미국 915MHz
```

### LoRa 물리 파라미터

| 파라미터 | 값 | 영향 |
|---------|-----|------|
| **SF (Spreading Factor)** | 7~12 | 높을수록 도달 거리↑, 데이터율↓, 전력↑ |
| **BW (Bandwidth)** | 125/250/500 kHz | 넓을수록 데이터율↑, 노이즈 민감↓ |
| **CR (Code Rate)** | 4/5~4/8 | 높을수록 오류 복구↑, 오버헤드↑ |
| **최대 페이로드** | 51~242 바이트 | SF에 따라 다름 |
| **에어타임** | SF7: ~50ms / SF12: ~1.5s | 한 패킷 전송 시간 |

```
[도달 거리 vs 데이터율]
SF7:  ~2km,  최대 5.5 kbps, 배터리 수명 짧음
SF12: ~15km, 최대 0.3 kbps, 배터리 수명 김

→ 실내/도심: SF7~9 / 농장·교외: SF10~12
```

---

## 2. LoRaWAN 네트워크 구조

```
[디바이스 (End Device)]
  LoRa 무선 전송 (단방향 위주)
        │
[게이트웨이 (Gateway)]          ← Dragino, RAK Wireless, Kerlink 등
  LoRa ↔ IP 변환, 패킷 포워딩
  Semtech Packet Forwarder 또는 Basics Station 프로토콜 사용
        │ (UDP 1700 또는 WebSocket)
[Network Server (ChirpStack)]
  중복 제거 (여러 게이트웨이 수신 통합)
  MAC 명령 처리 (ADR, 채널 설정)
  OTAA 조인 처리 (DevEUI, AppEUI, AppKey)
  페이로드 복호화 (AES-128 NwkSKey/AppSKey)
        │ (MQTT)
[Application Server (ChirpStack 내장 또는 외부)]
  페이로드 디코딩 (디바이스별 codec)
  MQTT 발행: application/{id}/device/{devEUI}/event/up
        │
[데이터 처리 파이프라인]  ← [[Node-RED]], [[MQTT]], 백엔드 서비스
```

---

## 3. 디바이스 인증 방식

### OTAA (Over-The-Air Activation) — 권장

```
[Join Request]
  디바이스 → Network Server
  DevEUI (고유 디바이스 ID, EUI-64)
  AppEUI / JoinEUI (애플리케이션 식별자)
  DevNonce (랜덤, 리플레이 방지)
  MIC (AppKey로 서명)

[Join Accept]
  Network Server → 디바이스
  AppNonce, NetID, DevAddr (세션 주소)
  NwkSKey (Network Session Key, AES-128) ← MAC 암호화
  AppSKey (Application Session Key, AES-128) ← 페이로드 암호화
  세션 키는 매 조인마다 갱신 → 보안↑

특징: 동적 세션 키, 재배포 유연, 권장 방식 ⭐
```

### ABP (Activation By Personalization) — 제한적 사용

```
DevAddr, NwkSKey, AppSKey 고정 하드코딩
→ 편리하지만 키 노출 시 교체 어려움
→ FCnt (Frame Counter) 리셋 문제
→ 산업 환경 일부에서만 사용
```

---

## 4. ChirpStack 설치 (Docker Compose)

```yaml
# docker-compose.yml
version: "3"
services:
  chirpstack:
    image: chirpstack/chirpstack:4
    command: -c /etc/chirpstack
    volumes:
      - ./chirpstack:/etc/chirpstack
    environment:
      - POSTGRESQL_DSN=postgresql://chirpstack:chirpstack@postgresql/chirpstack?sslmode=disable
      - REDIS_ADDR=redis:6379
    ports:
      - "8080:8080"   # Web UI / API
    depends_on:
      - postgresql
      - redis
      - mosquitto

  chirpstack-gateway-bridge:
    image: chirpstack/chirpstack-gateway-bridge:4
    ports:
      - "1700:1700/udp"   # Semtech UDP Packet Forwarder
    environment:
      - INTEGRATION__MQTT__EVENT_TOPIC_TEMPLATE=as923/gateway/{{ .GatewayID }}/event/{{ .EventType }}
      - INTEGRATION__MQTT__STATE_TOPIC_TEMPLATE=as923/gateway/{{ .GatewayID }}/state/{{ .StateType }}
      - INTEGRATION__MQTT__SERVER=tcp://mosquitto:1883
    depends_on:
      - mosquitto

  postgresql:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: chirpstack
      POSTGRES_PASSWORD: chirpstack
      POSTGRES_DB: chirpstack
    volumes:
      - postgresql-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

  mosquitto:
    image: eclipse-mosquitto:2
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config

volumes:
  postgresql-data:
  redis-data:
```

```bash
docker compose up -d
# ChirpStack UI: http://localhost:8080
# 초기 계정: admin / admin
```

---

## 5. ChirpStack 설정 순서

```
1. Network Server 등록 (자동 - 같은 인스턴스)
2. Service Profile 생성 (ADR 활성화, 최대 DR 설정)
3. Gateway 등록 (Gateway EUI 입력 - 게이트웨이 라벨에 표시)
4. Device Profile 생성
   - MAC 버전: LoRaWAN 1.0.3 또는 1.1
   - 지역: AS923 (한국)
   - 코덱 함수 (JavaScript): 바이너리 페이로드 디코딩
5. Application 생성 + MQTT Integration 활성화
6. Device 등록 (DevEUI, AppKey 입력)
7. 디바이스 전원 → OTAA Join 확인
```

### 페이로드 디코딩 함수 (ChirpStack Codec)

```javascript
// Device Profile → Codec (JavaScript)
// 바이너리 → JSON 변환 (ChirpStack에서 직접 실행)

function decodeUplink(input) {
    // input.bytes: Uint8Array, input.fPort: 포트 번호
    var buf = input.bytes;

    if (input.fPort === 1) {
        return {
            data: {
                temperature: ((buf[0] << 8) | buf[1]) / 100.0,  // 2바이트, 0.01°C
                humidity:    buf[2],                              // 1바이트, %
                battery:     (buf[3] / 254.0 * 100).toFixed(1), // 배터리 %
                rssi:        input.rxInfo[0].rssi,               // 신호 강도
                snr:         input.rxInfo[0].snr
            }
        };
    }
    return { errors: ["unknown fPort"] };
}

function encodeDownlink(input) {
    // Downlink: 서버 → 디바이스 (Class C 또는 Confirmed Down)
    return {
        bytes: [input.data.interval & 0xFF],
        fPort: 2
    };
}
```

---

## 6. MQTT 이벤트 토픽 구조

ChirpStack이 발행하는 MQTT 토픽:

```
application/{applicationID}/device/{devEUI}/event/up      ← Uplink (디바이스 → 서버)
application/{applicationID}/device/{devEUI}/event/join    ← Join 완료
application/{applicationID}/device/{devEUI}/event/ack     ← Confirmed 확인
application/{applicationID}/device/{devEUI}/event/txack   ← 다운링크 전송 확인
application/{applicationID}/device/{devEUI}/command/down  ← Downlink (서버 → 디바이스)
```

```json
// Uplink 페이로드 예시
{
  "deduplicationId": "uuid-xxx",
  "time": "2026-06-16T09:30:00.000Z",
  "deviceInfo": {
    "tenantId": "...",
    "applicationId": "app-001",
    "deviceName": "sensor-field-01",
    "devEui": "0102030405060708",
    "tags": {"location": "field-a", "type": "soil"}
  },
  "devAddr": "01020304",
  "adr": true,
  "dr": 3,
  "fCnt": 1234,
  "fPort": 1,
  "confirmed": false,
  "data": "ABCD",                         ← Base64 원시 바이트 (코덱 없을 때)
  "object": {                              ← 코덱 실행 결과 (JSON)
    "temperature": 23.5,
    "humidity": 65,
    "battery": "87.4"
  },
  "rxInfo": [{
    "gatewayId": "aabbccddeeff0011",
    "rssi": -87,
    "snr": 9.5,
    "location": {"latitude": 37.5, "longitude": 127.0}
  }]
}
```

---

## 7. 게이트웨이 하드웨어

| 제품 | 채널 수 | 가격대 | 특징 |
|------|--------|-------|------|
| **Dragino LG308** | 8채널 | ~$100 | 소형, 실외 방수, 초보자용 |
| **RAK7268** | 8채널 | ~$150 | Wi-Fi/LTE/이더넷 복합 |
| **Kerlink iFemtoCell** | 8채널 | ~$200 | 실내, 안정적 |
| **Multitech Conduit** | 8채널 | ~$500 | 엔터프라이즈, 방수 IP67 |
| **TTIG (Things Indoor)** | 8채널 | ~$70 | TTN 공식, 초간단 설치 |

---

## 8. 관련
- [[MQTT]] — ChirpStack → MQTT 발행 이후 처리
- [[Node-RED]] — MQTT 구독 + 페이로드 파싱 + 라우팅
- [[IoT-Architecture]] — ChirpStack을 포함한 전체 아키텍처
- [[IoT-Data-Ingestion]] — 다른 데이터 수집 방식과 비교
