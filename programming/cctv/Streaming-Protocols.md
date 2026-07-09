---
tags:
  - cctv
  - streaming
  - rtsp
  - rtmp
  - hls
  - webrtc
  - srt
created: 2026-06-16
---

# 스트리밍 프로토콜 — RTSP / RTMP / HLS / WebRTC / SRT

> [!summary] 한 줄 요약
> 각 프로토콜은 지연시간·호환성·인프라 비용 간 트레이드오프. CCTV 기본은 **RTSP**(낮은 지연, VMS 연동), 웹 브라우저 배포는 **HLS**(광범위 호환) 또는 **WebRTC**(실시간 < 1s), 방화벽 관통·장거리 링크는 **SRT**.

---

## 1. 프로토콜 비교 총람

| | RTSP | RTMP | HLS | WebRTC | SRT |
|--|------|------|-----|--------|-----|
| **지연** | 0.1~2초 | 0.5~3초 | 5~30초 | < 0.5초 | 1~4초 |
| **전송** | RTP/UDP (주로) | TCP | HTTP/TCP | UDP (DTLS) | UDP |
| **브라우저** | ❌ (플러그인) | ❌ (Flash 종료) | ✅ 네이티브 | ✅ 네이티브 | ❌ |
| **방화벽 관통** | 어려움 (포트 554) | 쉬움 (포트 1935/80) | 쉬움 (HTTP) | ICE/TURN으로 가능 | 어려움 |
| **코덱** | H.264/H.265 | H.264 (공식) | H.264/H.265 | VP8/VP9/H.264/AV1 | H.264/H.265/AV1 |
| **IP 카메라 지원** | ✅ 표준 | 일부 | 일부 | 드물게 | 드물게 |
| **녹화 용이성** | 보통 | 쉬움 | 쉬움 (TS 세그먼트) | 어려움 | 보통 |

---

## 2. RTSP (Real-Time Streaming Protocol)

### 개요

```
표준: RFC 2326 (RTSP 1.0), RFC 7826 (RTSP 2.0)
포트: 554 (RTSP), 8554 (대안)
전송: RTP over UDP (기본) 또는 TCP (방화벽 환경)
제어 프로토콜: RTSP (DESCRIBE, SETUP, PLAY, PAUSE, TEARDOWN)
```

### RTSP 연결 흐름

```
클라이언트                         서버 (카메라/MediaMTX)
    │                                    │
    │── OPTIONS rtsp://server/cam1 ──────▶│
    │◀── 200 OK (지원 메서드 목록) ───────│
    │                                    │
    │── DESCRIBE rtsp://server/cam1 ─────▶│  ← SDP 요청
    │◀── 200 OK + SDP ───────────────────│  ← 코덱, 포트 등 협상
    │                                    │
    │── SETUP .../track1 (RTP/AVP/UDP) ──▶│  ← 전송 방식 협상
    │◀── 200 OK (Session: xxxxx) ────────│
    │                                    │
    │── PLAY ────────────────────────────▶│  ← 스트리밍 시작
    │◀══════════ RTP 패킷 ════════════════│  ← 실제 영상 데이터
    │                                    │
    │── TEARDOWN ─────────────────────────▶│
```

### SDP (Session Description Protocol) 예시

```
v=0
o=- 0 0 IN IP4 192.168.1.100
s=CCTV Stream
c=IN IP4 0.0.0.0
t=0 0
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 profile-level-id=42e01e; packetization-mode=1;
          sprop-parameter-sets=Z0LgHtoCgPRA,aM4G4g==
a=control:track1
```

### RTSP URL 형식

```
rtsp://[username:password@]host[:port]/path
rtsp://admin:12345@192.168.1.100:554/Streaming/Channels/101

# 일반 IP 카메라 URL 패턴
Hikvision:  rtsp://admin:pass@ip/Streaming/Channels/101  (채널 1, 메인스트림)
                                /Streaming/Channels/102  (채널 1, 서브스트림)
Dahua:      rtsp://admin:pass@ip/cam/realmonitor?channel=1&subtype=0
Hanwha/QnV: rtsp://admin:pass@ip/profile1/media.smp
```

### RTP over TCP 강제 (방화벽 환경)

```
ffplay -rtsp_transport tcp rtsp://192.168.1.100:554/stream1
# 또는 MediaMTX 설정에서 rtspTransports: [tcp]
```

---

## 3. RTMP (Real-Time Messaging Protocol)

```
개발: Adobe (2002), 사양 공개 (2012)
포트: 1935 (RTMP), 443 (RTMPS/TLS), 80 (RTMPT/HTTP 터널링)
전송: TCP (신뢰성↑, 지연↑)
코덱: H.264 + AAC (공식), H.265 일부 지원 (비표준)

현황:
  - Flash 종료(2020)로 클라이언트 수신용으로는 사실상 사용 안 함
  - 하지만 **발행(push) 프로토콜**로 여전히 현역
  - OBS Studio, 스트리밍 소프트웨어 → RTMP → 미디어 서버 패턴
  - CCTV: IP 카메라 RTMP 출력 → MediaMTX 수신 → HLS/WebRTC 변환

RTMP → HLS 변환 (MediaMTX 내부):
  카메라 RTMP push → MediaMTX → HLS 세그먼트 생성 → 브라우저 재생
```

---

## 4. HLS (HTTP Live Streaming)

### 개요

```
개발: Apple (2009), IETF RFC 8216
전송: HTTP/HTTPS (CDN 친화적)
세그먼트: .ts 파일 (기본 6초) + .m3u8 플레이리스트
지연: 일반 HLS 6~30초 / Low-Latency HLS (LL-HLS) 2~5초

장점:
  - 모든 브라우저, iOS, Android 기본 지원
  - CDN으로 대규모 동시 시청자 처리
  - HTTP 방화벽 통과
  - 코덱 유연 (H.264/H.265/AV1 지원)

단점:
  - 높은 지연 (실시간 CCTV 모니터링 부적합)
  - 저장 공간 (세그먼트 파일 관리)
```

### m3u8 플레이리스트 구조

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:1523

#EXTINF:6.000000,
seg-1523.ts
#EXTINF:6.000000,
seg-1524.ts
#EXTINF:6.000000,
seg-1525.ts
```

### 어댑티브 비트레이트 (ABR) 마스터 플레이리스트

```m3u8
#EXTM3U
#EXT-X-VERSION:3

#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
low/stream.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
mid/stream.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
high/stream.m3u8
```

---

## 5. WebRTC

### 개요

```
표준: W3C/IETF (2011~)
전송: UDP (SRTP 암호화)
시그널링: 별도 채널 필요 (WebSocket, HTTP)
지연: < 500ms (실제 100~300ms)

장점:
  - 브라우저 네이티브, 플러그인 불필요
  - 초저지연 (실시간 양방향)
  - 내장 암호화 (DTLS/SRTP 필수)
  - P2P 가능

단점:
  - 인프라 복잡 (STUN/TURN 서버 필요)
  - 다수 시청자 확장 어려움 (SFU 필요)
  - 서버 CPU 부하 높음
```

### WebRTC 연결 수립 과정

```
브라우저 (Viewer)                  MediaMTX (Server)
       │                                  │
       │── WebSocket 시그널링 연결 ────────▶│
       │                                  │
       │── SDP Offer (지원 코덱, ICE) ─────▶│
       │◀── SDP Answer ──────────────────│
       │                                  │
       │[ICE 후보 수집]                    │
       │  STUN: 공인 IP:포트 확인          │
       │  TURN: NAT 관통 불가 시 릴레이    │
       │                                  │
       │── ICE 후보 교환 ─────────────────▶│
       │                                  │
       │[DTLS 핸드셰이크]                  │
       │── DTLS Client Hello ─────────────▶│
       │◀── DTLS Server Hello ────────────│
       │                                  │
       │◀══════ SRTP 영상 스트림 ══════════│
```

### STUN/TURN 설정 (MediaMTX)

```yaml
# mediamtx.yml
webrtc: yes
webrtcLocalUDPAddress: :8189
webrtcICEServers2:
  - urls: [stun:stun.l.google.com:19302]
  - urls: [turn:turn.example.com:3478]
    username: user
    password: pass
```

---

## 6. SRT (Secure Reliable Transport)

```
개발: Haivision (2017), 오픈소스 기부
전송: UDP 기반 (ARQ 재전송으로 신뢰성 확보)
포트: 기본 9710 (UDP)
암호화: AES-128/256

특징:
  - 인터넷 구간 장거리 스트리밍 최적화
  - 패킷 손실 자동 재전송 (ARQ)
  - 지연 허용치(latency) 설정으로 품질 보장
  - H.264/H.265/AV1 모두 지원
  - 방화벽 관통: Caller/Listener/Rendezvous 모드

용도:
  - 현장 중계 → 방송국 전송
  - 지점 간 원격 카메라 전송
  - 불안정한 네트워크 환경 (모바일 LTE 등)

FFmpeg SRT:
  # Caller 모드 (원격 서버에 연결)
  ffmpeg -re -i input.mp4 -c copy "srt://remote-server:9710?pkt_size=1316"

  # Listener 모드 (연결 수신 대기)
  ffmpeg -i "srt://0.0.0.0:9710?mode=listener" -c copy output.ts
```

---

## 7. CCTV 시나리오별 프로토콜 선택

```
[IP 카메라 → VMS(Video Management Software)]
  → RTSP (카메라 표준, 낮은 지연)

[CCTV → 웹 브라우저 실시간 모니터링]
  지연 5초 이상 허용: HLS (가장 호환 좋음)
  지연 1초 이내 필요: WebRTC (MediaMTX로 변환)

[멀티캠 동시 시청 (100명+)]
  → HLS + CDN (HTTP 계층 캐시 활용)

[야외 현장 → 본사 전송 (불안정 네트워크)]
  → SRT (패킷 손실 복구)

[OBS/방송 소프트웨어 → 서버]
  → RTMP push → 서버에서 HLS/WebRTC로 재배포

[전형적인 CCTV 서버 아키텍처]
  카메라(RTSP) → MediaMTX → RTSP(VMS) + HLS(웹) + WebRTC(모바일)
```

---

## 8. ONVIF — IP 카메라 표준 인터페이스

### ONVIF란

```
ONVIF (Open Network Video Interface Forum)
  - Axis, Bosch, Sony 공동 설립 (2008)
  - IP 감시 카메라의 상호운용성 표준
  - SOAP over HTTP 기반 디바이스 제어 API
  - 발견(WS-Discovery) + 설정 + 스트리밍 + PTZ 표준화

ONVIF 포트:
  SOAP API:     http://camera-ip:80/onvif/device_service
  RTSP 스트림:  rtsp://camera-ip:554/...
  WS-Discovery: UDP 멀티캐스트 239.255.255.250:3702
```

### ONVIF Profile 비교

| Profile | 버전 | 대상 | 핵심 기능 |
|---------|------|------|----------|
| **Profile S** | 2011 | IP 카메라·NVR | RTSP/RTP 스트리밍, H.264, PTZ, 오디오 — 가장 범용 |
| **Profile G** | 2013 | NVR·스토리지 | 엣지 녹화, 검색·재생, 스트리지 설정 |
| **Profile C** | 2013 | 출입통제 | 도어 잠금/해제, 자격증명, 이벤트 |
| **Profile Q** | 2015 | 빠른 설치 | 자동 발견, 초기 설정 간소화 |
| **Profile A** | 2018 | 출입통제 고급 | Profile C 확장, 크리덴셜 관리 |
| **Profile T** | 2018 | 고급 스트리밍 | H.265, 메타데이터 스트리밍, 이미지 분석 |
| **Profile M** | 2020 | 메타데이터·분석 | AI 이벤트, 물체 감지 메타데이터, HEVC |
| **Profile D** | 2021 | 주변장치 | 와이퍼, IR 조명 등 주변 기기 제어 |

### Profile S — RTSP 스트림 URL 획득

```xml
<!-- SOAP 요청: GetProfiles -->
POST http://192.168.1.100/onvif/media_service HTTP/1.1
Content-Type: application/soap+xml

<s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope">
  <s:Body>
    <trt:GetProfiles xmlns:trt="http://www.onvif.org/ver10/media/wsdl"/>
  </s:Body>
</s:Envelope>

<!-- 응답에서 ProfileToken 추출 후 GetStreamUri 호출 -->
<trt:GetStreamUri>
  <trt:StreamSetup>
    <tt:Stream>RTP-Unicast</tt:Stream>
    <tt:Transport><tt:Protocol>RTSP</tt:Protocol></tt:Transport>
  </trt:StreamSetup>
  <trt:ProfileToken>Profile_1</trt:ProfileToken>
</trt:GetStreamUri>

<!-- 응답 -->
<trt:Uri>rtsp://192.168.1.100:554/Streaming/Channels/101</trt:Uri>
```

### Profile T — H.265 + 메타데이터

```
Profile T 카메라 = H.265 스트리밍 지원
  - GetVideoSourceConfigurations2 (H.265 옵션 확인)
  - GetOSDOptions (OSD 텍스트/그래픽)
  - 메타데이터 스트림: 객체 경계 박스, 이벤트 데이터를 RTP로 전송

RTSP 멀티스트림 (Profile T):
  rtsp://camera/stream1   → H.265 메인 스트림
  rtsp://camera/stream2   → H.264 서브 스트림 (호환용)
  rtsp://camera/metadata  → 메타데이터 스트림 (AI 결과 등)
```

### Profile M — AI 분석 메타데이터

```
Profile M = AI 카메라 표준
  - 물체 감지 결과 (사람/차량/동물 분류)
  - 경계 박스 좌표 (ONVIF MetadataStream XSD)
  - 이벤트 알림 (tamper, line crossing, intrusion)

메타데이터 구조:
<tt:MetadataStream>
  <tt:VideoAnalytics>
    <tt:Frame UtcTime="2026-06-16T09:30:00Z">
      <tt:Object ObjectId="1">
        <tt:Appearance>
          <tt:Class>
            <tt:Type Likelihood="0.95">Human</tt:Type>
          </tt:Class>
          <tt:BoundingBox left="0.3" top="0.2" right="0.5" bottom="0.8"/>
        </tt:Appearance>
      </tt:Object>
    </tt:Frame>
  </tt:VideoAnalytics>
</tt:MetadataStream>
```

---

## 9. PTZ (Pan-Tilt-Zoom) 제어

### PTZ 제어 방식

```
[프로토콜 기반]
  - ONVIF PTZ Service (SOAP/HTTP) — 표준, 브랜드 무관
  - Pelco-D / Pelco-P (RS-485 시리얼) — 아날로그 PTZ 레거시
  - Visca over IP (Sony 계열) — Sony/Canon 카메라

[이동 명령 종류]
  - ContinuousMove: 속도 벡터로 지속 이동 (Stop 신호 필요)
  - AbsoluteMove: 절대 좌표 이동 (Pan -1.0~1.0, Tilt -1.0~1.0, Zoom 0~1.0)
  - RelativeMove: 현재 위치 기준 상대 이동
  - GotoPreset: 사전 저장 위치로 즉시 이동
```

### ONVIF PTZ SOAP API

```java
// Java — ONVIF PTZ 제어 (onvif4j 또는 직접 SOAP 호출)
// Maven: com.github.RootSoft:onvif4j 또는 직접 구현

// SOAP 요청 — ContinuousMove (왼쪽으로 이동)
POST http://192.168.1.100/onvif/ptz_service

<tptz:ContinuousMove>
  <tptz:ProfileToken>Profile_1</tptz:ProfileToken>
  <tptz:Velocity>
    <tt:PanTilt x="-0.5" y="0" space="...PanTiltGenericSpace"/>
    <tt:Zoom x="0" space="...ZoomGenericSpace"/>
  </tptz:Velocity>
  <tptz:Timeout>PT2S</tptz:Timeout>  <!-- 2초 후 자동 정지 -->
</tptz:ContinuousMove>

// AbsoluteMove — 정중앙 위치로 이동
<tptz:AbsoluteMove>
  <tptz:ProfileToken>Profile_1</tptz:ProfileToken>
  <tptz:Position>
    <tt:PanTilt x="0" y="0"/>   <!-- 정중앙 -->
    <tt:Zoom x="0.5"/>          <!-- 중간 줌 -->
  </tptz:Position>
</tptz:AbsoluteMove>

// GotoPreset — 저장된 위치 1번으로 이동
<tptz:GotoPreset>
  <tptz:ProfileToken>Profile_1</tptz:ProfileToken>
  <tptz:PresetToken>1</tptz:PresetToken>
</tptz:GotoPreset>
```

### Spring 기반 PTZ 클라이언트

```java
@Service
@RequiredArgsConstructor
public class PtzControlService {

    private final RestTemplate restTemplate;

    private static final String PTZ_SERVICE = "/onvif/ptz_service";

    public void continuousMove(String cameraIp, double panSpeed, double tiltSpeed) {
        String url = "http://" + cameraIp + PTZ_SERVICE;
        String soapBody = """
            <s:Envelope xmlns:s="http://www.w3.org/2003/05/soap-envelope"
                        xmlns:tptz="http://www.onvif.org/ver20/ptz/wsdl"
                        xmlns:tt="http://www.onvif.org/ver10/schema">
              <s:Body>
                <tptz:ContinuousMove>
                  <tptz:ProfileToken>Profile_1</tptz:ProfileToken>
                  <tptz:Velocity>
                    <tt:PanTilt x="%f" y="%f"/>
                    <tt:Zoom x="0"/>
                  </tptz:Velocity>
                  <tptz:Timeout>PT3S</tptz:Timeout>
                </tptz:ContinuousMove>
              </s:Body>
            </s:Envelope>
            """.formatted(panSpeed, tiltSpeed);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.TEXT_XML);
        // Digest Auth (ONVIF 표준)
        headers.set("Authorization", buildDigestAuth(cameraIp, "admin", "pass"));

        restTemplate.postForEntity(url, new HttpEntity<>(soapBody, headers), String.class);
    }

    public void stop(String cameraIp) {
        // PanTilt x="0" y="0" 로 ContinuousMove 호출 또는 Stop 명령
    }

    public void gotoPreset(String cameraIp, int presetNumber) {
        // GotoPreset SOAP 호출
    }
}
```

### PTZ 투어 (자동 순환)

```java
@Component
@RequiredArgsConstructor
public class PtzTourScheduler {

    private final PtzControlService ptzService;
    private static final String CAM_IP = "192.168.1.100";

    @Scheduled(fixedDelay = 30000) // 30초마다 다음 프리셋으로
    public void rotateTour() {
        int preset = (currentPreset.getAndIncrement() % 4) + 1; // 프리셋 1~4 순환
        ptzService.gotoPreset(CAM_IP, preset);
    }

    private final AtomicInteger currentPreset = new AtomicInteger(0);
}
```

---

## 10. 관련
- [[Video-Fundamentals]] — 코덱, 색상 포맷, 컨테이너
- [[MediaMTX]] — RTSP/RTMP/HLS/WebRTC 동시 제공 서버
- [[FFmpeg]] — 프로토콜 간 변환, 트랜스코딩
- [[Spring-CCTV]] — Spring 기반 JavaCV vs FFmpeg 멀티프로세스 처리 비교
