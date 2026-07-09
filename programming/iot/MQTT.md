---
tags:
  - iot
  - mqtt
  - protocol
  - broker
  - messaging
created: 2026-06-16
---

# MQTT — 프로토콜 심화

> [!summary] 한 줄 요약
> MQTT는 **TCP 위의 Pub/Sub 메시징 프로토콜**. ISO 표준(ISO/IEC 20922). 헤더가 2바이트로 시작하는 극도로 경량이며, 불안정한 네트워크와 저전력 디바이스를 위해 설계됐다.

---

## 1. 패킷 구조

```
[Fixed Header — 최소 2바이트]
  Byte 1: [패킷 타입 4bit][플래그 4bit]
  Byte 2~5: Remaining Length (가변 길이 인코딩, 최대 256MB)

[Variable Header] — 패킷 타입별 다름
[Payload]         — 패킷 타입별 다름
```

### MQTT 패킷 타입

| 값 | 타입 | 방향 | 설명 |
|----|------|------|------|
| 1 | CONNECT | C→S | 연결 요청 |
| 2 | CONNACK | S→C | 연결 응답 |
| 3 | PUBLISH | 양방향 | 메시지 발행 |
| 4 | PUBACK | 양방향 | QoS 1 확인 |
| 5 | PUBREC | 양방향 | QoS 2 수신확인 |
| 6 | PUBREL | 양방향 | QoS 2 해제 |
| 7 | PUBCOMP | 양방향 | QoS 2 완료 |
| 8 | SUBSCRIBE | C→S | 구독 요청 |
| 9 | SUBACK | S→C | 구독 응답 |
| 12 | PINGREQ | C→S | 킵얼라이브 |
| 14 | DISCONNECT | C→S | 연결 종료 |

---

## 2. 연결 수명주기

```
[CONNECT]
  clientId: "device-001"         ← 브로커에서 유일해야 함 (중복 시 이전 연결 강제 종료)
  cleanSession: false            ← false: 재연결 시 미수신 메시지 수신
  keepAlive: 60                  ← 초 단위. 이 시간 내 패킷 없으면 브로커가 연결 종료
  willTopic: "status/device-001" ← LWT 토픽
  willPayload: {"online":false}  ← LWT 페이로드 (비정상 종료 시 브로커가 발행)
  willQoS: 1
  willRetain: true               ← 구독자가 나중에 연결해도 최근 상태 수신

[CONNACK]
  returnCode: 0   ← 0=성공, 1=프로토콜 불일치, 2=clientId 거부,
                     3=서버 불가, 4=인증 실패, 5=인가 실패

[PINGREQ / PINGRESP] — keepAlive 마다 교환 (좀비 연결 탐지)

[DISCONNECT] → 브로커에 LWT 취소 알림 후 정상 종료
비정상 종료  → keepAlive 초과 시 브로커 감지 → LWT 자동 발행
```

---

## 3. QoS (Quality of Service) 상세

### QoS 0 — At Most Once (Fire and Forget)

```
Publisher   ─── PUBLISH ──►   Broker   ─── PUBLISH ──►   Subscriber
            (확인 없음)                  (확인 없음)
```
- 가장 빠름, 오버헤드 없음
- 패킷 손실 허용: 최신 값만 의미있는 센서 데이터
- 브로커 재시작 시 전달 보장 없음

### QoS 1 — At Least Once

```
Publisher   ─── PUBLISH (packetId=1) ──►   Broker
            ◄── PUBACK  (packetId=1) ───
            (PUBACK 받을 때까지 재전송)
```
- 최소 1회 전달 보장
- **중복 가능**: 구독자가 idempotent 처리 필요
- 가장 일반적인 IoT 선택 ⭐

### QoS 2 — Exactly Once

```
Publisher   ─── PUBLISH (id=1) ──►   Broker
            ◄── PUBREC  (id=1) ───
            ─── PUBREL  (id=1) ──►
            ◄── PUBCOMP (id=1) ───   (4-way handshake)
```
- 정확히 1회 전달 보장
- 오버헤드 최대 (4번 메시지 교환)
- 결제·제어 명령·중복 불가 이벤트

---

## 4. Retained Message & LWT

### Retained Message

```bash
# retain=true로 발행 → 브로커가 토픽별 최신 1개 보관
mosquitto_pub -h broker -t "status/device-001" \
  -m '{"online":true,"fw":"1.2.3"}' --retain

# 새 구독자가 subscribe하는 즉시 보관된 최신 메시지 수신
# → 디바이스 상태 레지스트리로 활용
mosquitto_sub -h broker -t "status/#"
# 즉시: status/device-001 {"online":true,...}  ← 이전에 발행됐어도 수신
```

### LWT (Last Will and Testament) 활용 패턴

```
연결 시: CONNECT + willTopic="status/device-001", willPayload={"online":false}, retain=true
정상 동작: PUBLISH status/device-001 {"online":true} retain=true  ← 주기적 갱신
비정상 종료: 브로커가 keepAlive 초과 감지 → willPayload 자동 발행
             → 구독자: status 토픽에서 {"online":false} 수신 → 알림 발생
```

---

## 5. MQTT 5.0 주요 신기능

MQTT 3.1.1에서 MQTT 5.0으로의 주요 변화:

```
[이유 코드 (Reason Code)]
  CONNACK에 상세 오류 이유 포함
  "인증 실패"가 아닌 "잘못된 사용자명" 구분 가능

[메시지 만료 (Message Expiry)]
  PUBLISH에 TTL 설정 → 브로커가 오래된 메시지 자동 삭제
  { "Message-Expiry-Interval": 3600 }  ← 1시간 후 만료

[공유 구독 (Shared Subscriptions)]
  $share/group-name/topic  ← 그룹 내 1개 구독자에만 전달 (로드 밸런싱)
  여러 소비자가 처리 부하를 나눌 수 있음

[요청/응답 패턴]
  PUBLISH에 Response-Topic + Correlation-Data 포함
  → MQTT 위에서 RPC 패턴 구현 가능

[User Properties]
  헤더에 임의 키-값 쌍 추가 (HTTP 헤더와 유사)
```

---

## 6. 보안 설정

```
[전송 계층]
  TLS 1.3: MQTTS (포트 8883)
  mosquitto.conf:
    listener 8883
    cafile   /etc/mosquitto/ca.crt
    certfile /etc/mosquitto/server.crt
    keyfile  /etc/mosquitto/server.key
    require_certificate true   ← 클라이언트 인증서 필수 (mTLS)

[인증]
  패스워드 파일: mosquitto_passwd -c /etc/mosquitto/passwd username
  OAuth2/JWT: EMQX의 JWT 플러그인
  ACL: 토픽별 퍼블리시/구독 권한

[ACL 예시 (mosquitto ACL 파일)]
  user device-001
  topic write sensors/device-001/#    ← 자기 토픽만 발행 가능
  topic read  cmd/device-001/#        ← 자기 명령만 수신

  user backend-server
  topic read  sensors/#               ← 모든 센서 구독
  topic write cmd/#                   ← 모든 명령 발행
```

---

## 7. EMQX 고성능 브로커 설정 (Docker)

```yaml
# docker-compose.yml
services:
  emqx:
    image: emqx/emqx:5.8
    ports:
      - "1883:1883"    # MQTT
      - "8883:8883"    # MQTTS
      - "8083:8083"    # MQTT over WebSocket
      - "8084:8084"    # MQTT over WSS
      - "18083:18083"  # Dashboard (admin:public)
    volumes:
      - emqx-data:/opt/emqx/data
      - ./emqx.conf:/opt/emqx/etc/emqx.conf
    environment:
      EMQX_NODE__NAME: emqx@127.0.0.1
```

```bash
# 클러스터 상태 확인
docker exec emqx emqx_ctl status
docker exec emqx emqx_ctl broker stats
docker exec emqx emqx_ctl clients list

# 토픽 메시지 디버깅
docker exec emqx emqx_ctl subscriptions list
mosquitto_pub -h localhost -t "test/topic" -m "hello" -u user -P pass
mosquitto_sub -h localhost -t "test/#" -u user -P pass
```

---

## 8. 관련
- [[IoT-Data-Ingestion]] — 전체 프로토콜 비교
- [[LoRaWAN-ChirpStack]] — LoRa → ChirpStack → MQTT 흐름
- [[Node-RED]] — MQTT 수신 후 파싱·변환
- [[IoT-Architecture]] — 전체 아키텍처 설계
