---
tags:
  - iot
  - architecture
  - chirpstack
  - node-red
  - mqtt
  - broker
created: 2026-06-16
---

# IoT 전체 아키텍처 — ChirpStack → MQTT → Node-RED → 소프트웨어 브로커

> [!summary] 한 줄 요약
> LoRa 디바이스의 데이터가 서비스 백엔드에 도달하기까지의 전체 파이프라인. **ChirpStack**이 LoRa 네트워크를 처리하고, **MQTT**를 통해 **Node-RED**로 전달되며, Node-RED가 파싱·변환 후 **소프트웨어 브로커(EMQX/Mosquitto)**를 거쳐 다양한 소비자에게 분배한다.

---

## 1. 전체 데이터 플로우

```
[LoRaWAN 디바이스]           [TCP 직접 연결 디바이스]    [HTTP 디바이스]    [MQTT 직접 디바이스]
  토양·수위·온습도 센서          산업용 PLC, 계량기         모바일·REST         기존 센서 노드
  배터리 구동, Class A           TCP 소켓                  POST /sensor/data   raw/device/+ 발행
        │ LoRa 무선                     │                       │                    │
        ▼                              │                       │                    │
[LoRa Gateway]                         │                       │                    │
  Semtech Packet Forwarder              │                       │                    │
  UDP 1700                             │                       │                    │
        │                              │                       │                    │
        ▼                              │                       │                    │
[ChirpStack GW Bridge]                 │                       │                    │
[ChirpStack NS/AS]                     │                       │                    │
  OTAA 인증, AES 복호화                 │                       │                    │
  Codec(JS) → JSON                      │                       │                    │
  MQTT publish: application/#/event/up  │                       │                    │
        │                              │                       │                    │
        ▼                              ▼                       ▼                    ▼
  [Raw Mosquitto]             ┌──────────────────────────────────────────────────────┐
  (ChirpStack 내부 버스)       │              Node-RED  (포트 1880)                   │
        │                     │  ┌──────────────────────────────────────────────┐    │
        └─────────────────────▶  │  Source Router + 단일 Normalizer 함수          │    │
           mqtt in             │  │  LoRaWAN / TCP / UDP / HTTP / MQTT → 통일 JSON │    │
           app/+/device/+/up   │  └──────────────────────────────────────────────┘    │
                               │               │                                      │
                               │     ┌─────────┴─────────┐                           │
                               │     ▼                   ▼                           │
                               │  [threshold check]   [mqtt out]                     │
                               │  이상값 → Telegram    topic: iot/processed/{loc}/{id}│
                               └──────────────────────────────────────────────────────┘
                                                          │
                                                          ▼
                                          [소프트웨어 MQTT 브로커 — EMQX]
                                            포트 1883/8883, ACL 권한 분리
                                                          │
                                       ┌──────────────────┼──────────────────┐
                                       ▼                  ▼                  ▼
                               [Spring Boot]       [TimescaleDB]         [Grafana]
                               비즈니스 로직        시계열 저장           대시보드
```

---

## 2. 컴포넌트별 역할 요약

| 컴포넌트 | 역할 | 기술 |
|---------|------|------|
| **LoRa Gateway** | 무선 → IP 변환 | Dragino, RAK Wireless |
| **ChirpStack GW Bridge** | UDP Packet Forwarder → MQTT | Go, Docker |
| **ChirpStack NS/AS** | 네트워크 서버, 코덱, 인증 | Go, PostgreSQL, Redis |
| **Raw MQTT Broker** | ChirpStack 내부 메시지 버스 | Mosquitto (경량) |
| **Node-RED** | 파싱·변환·라우팅·알림 | Node.js, Visual Flow |
| **소프트웨어 MQTT 브로커** | 정규화 데이터 분배 허브 | EMQX (고성능, 클러스터) |
| **Spring Backend** | API, 비즈니스 로직 | Java, Spring Boot |
| **TimescaleDB** | 시계열 데이터 저장 | PostgreSQL + TimescaleDB |

---

## 3. Docker Compose — 전체 스택

```yaml
version: "3.8"

services:
  # ── 1. LoRaWAN 레이어 ─────────────────────────────────────────

  chirpstack-gateway-bridge:
    image: chirpstack/chirpstack-gateway-bridge:4
    ports:
      - "1700:1700/udp"         # Semtech UDP PF
    environment:
      - INTEGRATION__MQTT__SERVER=tcp://mosquitto-raw:1883
      - INTEGRATION__MQTT__EVENT_TOPIC_TEMPLATE=as923/gateway/{{ .GatewayID }}/event/{{ .EventType }}

  chirpstack:
    image: chirpstack/chirpstack:4
    command: -c /etc/chirpstack
    volumes:
      - ./chirpstack-config:/etc/chirpstack
    ports:
      - "8080:8080"
    environment:
      - POSTGRESQL_DSN=postgresql://cs:cs@postgresql/chirpstack?sslmode=disable
      - REDIS_ADDR=redis:6379
      - INTEGRATION__MQTT__SERVER=tcp://mosquitto-raw:1883
    depends_on: [postgresql, redis, mosquitto-raw]

  # ── 2. Raw MQTT 브로커 (ChirpStack 내부용) ─────────────────────

  mosquitto-raw:
    image: eclipse-mosquitto:2
    volumes:
      - ./mosquitto-raw/mosquitto.conf:/mosquitto/config/mosquitto.conf
    # 외부 노출 불필요 (Node-RED와 같은 내부 네트워크)

  # ── 3. Node-RED 미들웨어 ──────────────────────────────────────

  node-red:
    image: nodered/node-red:latest
    ports:
      - "1880:1880"
    volumes:
      - node-red-data:/data
      - ./node-red/flows.json:/data/flows.json
    environment:
      - RAW_MQTT_URL=tcp://mosquitto-raw:1883
      - TARGET_MQTT_URL=tcp://emqx:1883
      - TARGET_MQTT_USER=nodered
      - TARGET_MQTT_PASS=nodered_pass
      - DB_URL=postgresql://iot:iot@timescaledb/iotdb
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
    depends_on: [mosquitto-raw, emqx]

  # ── 4. 소프트웨어 MQTT 브로커 (배포 허브) ──────────────────────

  emqx:
    image: emqx/emqx:5.8
    ports:
      - "1883:1883"     # MQTT
      - "8883:8883"     # MQTTS
      - "8083:8083"     # MQTT over WS
      - "18083:18083"   # Dashboard
    volumes:
      - emqx-data:/opt/emqx/data
      - ./emqx/acl.conf:/opt/emqx/etc/acl.conf

  # ── 5. 저장·분석 레이어 ──────────────────────────────────────

  timescaledb:
    image: timescale/timescaledb:latest-pg16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: iot
      POSTGRES_PASSWORD: iot
      POSTGRES_DB: iotdb
    volumes:
      - timescaledb-data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql

  # ── 인프라 ───────────────────────────────────────────────────

  postgresql:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: cs
      POSTGRES_PASSWORD: cs
      POSTGRES_DB: chirpstack

  redis:
    image: redis:7-alpine

volumes:
  node-red-data:
  emqx-data:
  timescaledb-data:
```

---

## 4. TimescaleDB 스키마

```sql
-- init.sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE sensor_readings (
    time        TIMESTAMPTZ NOT NULL,
    device_eui  VARCHAR(16) NOT NULL,
    device_name TEXT,
    location    TEXT,
    temperature DOUBLE PRECISION,
    humidity    DOUBLE PRECISION,
    battery     DOUBLE PRECISION,
    rssi        INTEGER,
    snr         DOUBLE PRECISION,
    raw_payload JSONB
);

SELECT create_hypertable('sensor_readings', by_range('time'));
CREATE INDEX ON sensor_readings (device_eui, time DESC);

-- 연속 집계 (1시간 단위)
CREATE MATERIALIZED VIEW sensor_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
       device_eui, location,
       avg(temperature) AS avg_temp,
       avg(humidity)    AS avg_humidity,
       min(battery)     AS min_battery
FROM sensor_readings
GROUP BY bucket, device_eui, location;

-- 90일 보존
SELECT add_retention_policy('sensor_readings', INTERVAL '90 days');
```

---

## 5. EMQX ACL — 소비자 권한 분리

```
# /opt/emqx/etc/acl.conf

# Node-RED: iot/processed/# 발행만 허용
{allow, {user, "nodered"}, publish, ["iot/processed/#"]}.
{allow, {user, "nodered"}, subscribe, []}.

# Spring Backend: iot/processed/# 구독만 허용
{allow, {user, "spring-backend"}, subscribe, ["iot/processed/#"]}.

# Grafana/InfluxDB 연동: 읽기 전용
{allow, {user, "grafana-agent"}, subscribe, ["iot/processed/#"]}.

# 관리자: 전체
{allow, {user, "admin"}, pubsub, ["#"]}.

# 기본: 모두 거부
{deny, all}.
```

---

## 6. 장애 대응 & 주요 고려사항

```
[Node-RED 장애 시]
  ChirpStack → Raw MQTT에 메시지 쌓임 (Mosquitto 큐)
  Node-RED 재시작 → cleanSession:false → 미처리 메시지 재처리
  → 중복 처리 가능: TimescaleDB ON CONFLICT DO NOTHING 필수

[EMQX 장애 시]
  Node-RED의 MQTT out 재시도 (persistent session)
  EMQX 클러스터 3노드 구성으로 HA 확보

[시계열 DB 장애 시]
  Node-RED → 파일 버퍼 노드로 로컬 임시 저장
  복구 후 재전송 (배치 INSERT)

[게이트웨이 통신 두절]
  ChirpStack: 게이트웨이 오프라인 감지 (Last Seen 기준)
  Node-RED LWT: 연결 모니터링 + Telegram 알림
```

---

## 7. 보안 체크리스트

```
✅ ChirpStack API: HTTPS + API Token 인증
✅ Raw MQTT (Mosquitto): 내부 네트워크만 노출, 외부 접근 금지
✅ Node-RED UI: 리버스 프록시 + BasicAuth 또는 VPN 접근만
✅ EMQX: TLS 8883, 클라이언트별 ACL
✅ TimescaleDB: 내부 네트워크, 방화벽 5432 차단
✅ 비밀값: Docker Secret 또는 K8s Secret (평문 환경변수 금지)
✅ LoRaWAN OTAA: AppKey 128비트 랜덤, 디바이스별 고유
```

---

## 8. 관련
- [[LoRaWAN-ChirpStack]] — ChirpStack 설치·설정·코덱 상세
- [[MQTT]] — MQTT 프로토콜, EMQX 브로커 설정
- [[Node-RED]] — 플로우 노드 상세, 파싱 패턴
- [[IoT-Data-Ingestion]] — 다른 수집 방식 비교
- [[../../infra/database/TimescaleDB]] — 시계열 저장소
