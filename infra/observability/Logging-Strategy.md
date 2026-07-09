---
tags:
  - observability
  - logging
  - audit-log
  - compliance
  - architecture
created: 2026-06-16
---

# 로깅 전략 — 중요성·감사로그·단일/다중화 아키텍처

> [!summary] 한 줄 요약
> 로그는 **장애 복구, 보안 사고 조사, 법적 의무**의 핵심이다. 단일 서버는 로컬 로그로 충분하지만, 다중화(MSA/K8s) 환경에서는 **중앙집중식 로그**가 필수다. 서버가 죽으면 로컬 로그도 사라진다.

---

## 1. 왜 로그가 중요한가

```
장애 발생 → 어디서 무슨 일이 일어났는가?
보안 침해 → 누가 언제 무엇을 접근했는가?
성능 저하 → 어떤 요청이 느렸는가?
규제 감사 → 개인정보를 누가 조회했는가?

↑ 모두 로그 없이는 대답 불가능
```

### 로그가 없을 때 발생하는 문제

| 상황 | 로그 없을 때 | 로그 있을 때 |
|------|------------|------------|
| 서버 장애 | "왜 죽었는지 모름" | 에러 스택트레이스·OOM·OutOfMemory 확인 |
| 개인정보 유출 | 유출 경로 불명 → 신고 불가 | 접근 IP·계정·시간 재구성 가능 |
| API 오용 | 탐지 불가 | 특정 IP 비정상 호출 패턴 감지 |
| 법원 요청 | 제출 불가 | 타임스탬프·행위 증명 가능 |
| 규제 과태료 | 입증 불가 → 최대 과태료 | 통제 적용 증명 → 감경 |

---

## 2. 감사 로그 (Audit Log) 표준

### 2.1 개인정보보호법 / ISMS-P 요구사항

```
개인정보보호법 제29조: 개인정보 처리 시스템에 대한 접근 기록 보관
  → 최소 6개월 보관 (법적 의무)
  → 접속자 계정, 접속 일시, 접속 IP, 처리한 개인정보 항목

ISMS-P 인증기준 2.9.4 로그 및 접속기록 관리:
  → 서버/DB/응용프로그램 로그 1년 이상 보관
  → 개인정보 처리 로그 별도 분리 보관
  → 로그 위·변조 방지 조치
```

### 2.2 감사 로그 필수 필드

```json
{
  "event_time":    "2026-06-16T09:30:00.123Z",   // 타임스탬프 (UTC)
  "event_type":    "DATA_ACCESS",                 // 이벤트 분류
  "actor_id":      "user-7890",                   // 행위자 ID
  "actor_ip":      "203.0.113.5",                 // 접근 IP
  "actor_role":    "OPERATOR",                    // 권한 레벨
  "resource_type": "PersonalInfo",                // 접근 리소스 유형
  "resource_id":   "customer-1234",               // 리소스 식별자
  "action":        "READ",                        // 수행 행위
  "result":        "SUCCESS",                     // 결과
  "data_fields":   ["name", "phone", "email"],    // 접근한 개인정보 항목
  "request_id":    "req-abc123",                  // 요청 추적 ID
  "session_id":    "sess-xyz789",                 // 세션 ID
  "service":       "member-service"               // 서비스명
}
```

### 2.3 이벤트 분류 체계

| 이벤트 타입 | 설명 | 보관 기간 |
|------------|------|----------|
| `AUTH_LOGIN` / `AUTH_LOGOUT` | 로그인·로그아웃 | 1년 |
| `AUTH_FAIL` | 인증 실패 (무차별 대입 탐지용) | 1년 |
| `DATA_ACCESS` | 개인정보 조회 | 5년 |
| `DATA_MODIFY` | 개인정보 수정·삭제 | 5년 |
| `ADMIN_ACTION` | 관리자 행위 | 5년 |
| `EXPORT` | 대량 데이터 추출 | 5년 |
| `PERMISSION_CHANGE` | 권한 변경 | 5년 |
| `SYSTEM_ERROR` | 시스템 오류 | 1년 |

### 2.4 Spring Boot 감사 로그 구현

```java
// 감사 이벤트 도메인 객체
public record AuditEvent(
    Instant eventTime,
    String  eventType,
    String  actorId,
    String  actorIp,
    String  resourceType,
    String  resourceId,
    String  action,
    String  result,
    List<String> dataFields,
    String  requestId
) {}

// 감사 로그 서비스
@Service
@RequiredArgsConstructor
public class AuditLogger {
    private static final Logger auditLog =
        LoggerFactory.getLogger("AUDIT");   // 전용 로거 채널

    public void log(AuditEvent event) {
        // 감사 로그 전용 MDC
        MDC.put("auditEvent", event.eventType());
        try {
            auditLog.info("{}", toJson(event));   // JSON으로 직렬화
        } finally {
            MDC.clear();
        }
    }
}

// AOP로 서비스 레이어 자동 감사
@Aspect
@Component
@RequiredArgsConstructor
public class AuditAspect {
    private final AuditLogger auditLogger;

    @Around("@annotation(Auditable)")
    public Object audit(ProceedingJoinPoint jp) throws Throwable {
        var annotation = ((MethodSignature) jp.getSignature())
            .getMethod().getAnnotation(Auditable.class);
        try {
            Object result = jp.proceed();
            auditLogger.log(buildEvent(jp, annotation, "SUCCESS"));
            return result;
        } catch (Exception e) {
            auditLogger.log(buildEvent(jp, annotation, "FAIL"));
            throw e;
        }
    }
}

// 사용 예
@GetMapping("/members/{id}")
@Auditable(eventType = "DATA_ACCESS", resourceType = "PersonalInfo",
           dataFields = {"name", "phone", "email"})
public MemberResponse getMember(@PathVariable Long id) { ... }
```

---

## 3. 단일 서버 vs 다중화 서버 로깅 아키텍처

### 3.1 단일 서버 (소규모)

```
[단일 서버]
  앱 프로세스
    └─ /var/log/app/app.log       ← 파일 직접 기록
    └─ /var/log/app/audit.log
    └─ journald (systemd)

관리 방법:
  - logrotate: 일별 압축, 최대 90일 보관
  - tail -f / grep / awk 로 직접 분석
  - 백업: cron + scp → 원격 저장소
```

```ini
# /etc/logrotate.d/myapp
/var/log/app/*.log {
    daily               # 매일 롤오버
    rotate 90           # 90개 보관
    compress            # gzip 압축
    delaycompress       # 직전 파일은 압축 미룸 (tail -f 충돌 방지)
    missingok           # 파일 없어도 오류 무시
    notifempty          # 비어있으면 롤오버 안 함
    postrotate
        systemctl reload myapp   # 앱에 파일 교체 알림
    endscript
}
```

**문제점**: 서버가 1대면 서버가 죽으면 로그도 접근 불가. 디스크 장애 시 로그 소실.

### 3.2 다중화 서버에서 중앙집중식 로그가 필요한 이유

```
[분산 환경 — 로그가 각 서버에 분산]

서버 A: order-service   → /var/log/order.log  ← 장애 시 접근 불가
서버 B: order-service   → /var/log/order.log  ← 두 파일 비교 불편
서버 C: payment-service → /var/log/payment.log
서버 D: member-service  → /var/log/member.log

문제:
  1. 하나의 요청이 A→C→D를 거치는데 각 서버 로그를 수동으로 합쳐야 함
  2. 서버 A가 죽으면 그 서버의 로그에 접근 불가 (죽기 직전 로그가 가장 중요!)
  3. A와 B의 타임스탬프가 1초 차이나면 시간순 정렬 불가
  4. 10개 서버를 동시에 grep하는 것은 불가능
  5. 오토스케일링: 인스턴스가 사라지면 로그도 사라짐 (K8s Pod evict)
```

```
[중앙집중식 로그 — 해결]

서버 A, B, C, D
     │ │ │ │  (에이전트가 실시간 전송 — 서버 죽기 전에)
     ▼ ▼ ▼ ▼
  [중앙 로그 스토어]    ← 단일 검색 포인트
  (Loki / Elasticsearch)
     │
  단일 쿼리로 전체 서버 로그 통합 조회
  요청 ID로 A→C→D 흐름 추적
  서버 A가 죽어도 이미 전송된 로그는 안전
```

### 3.3 K8s 환경에서 로컬 로그의 치명적 문제

```
[K8s Pod — 로컬 로그의 위험]

Pod A (order-service, node-1)
  └─ 컨테이너 내부 /tmp/app.log   ← Pod 삭제 시 즉시 소실
  └─ 노드의 /var/log/containers/  ← 노드 재시작·교체 시 소실

상황:
  OOM으로 Pod Evict → 새 Pod 생성 → 이전 Pod 로그 삭제
  노드 업그레이드 → 노드 교체 → 로그 소실
  오토스케일링 다운 → 인스턴스 종료 → 로그 소실

→ 중앙 로그가 없으면 "Pod가 왜 죽었는지" 영구 불명
```

---

## 4. 중앙집중식 로그 아키텍처 설계

### 4.1 레이어별 역할

```
[앱 레이어]
  구조화 로그 (JSON) → stdout
  감사 로그 → 별도 스트림 or 필드

[수집 레이어]
  Alloy / Filebeat (DaemonSet) → 로컬 → 중앙 저장소로 전송
  버퍼링: 중앙 저장소 장애 시 로컬 큐 유지 (데이터 유실 방지)

[저장 레이어]
  Loki (저비용, K8s 네이티브)
  Elasticsearch (풀텍스트 검색, SIEM)
  S3 / GCS (장기 아카이브, 규제 보관)

[분석 레이어]
  Grafana / Kibana — 대시보드·알림
  감사팀 전용 뷰 — 개인정보 접근 로그 분리
```

### 4.2 로그 분리 전략

```
하나의 로그 스트림에 전부 넣지 말 것 → 접근 제어·보관 기간 다름

[스트림 분리]
  앱 로그    → loki{job="app-logs"}     보관: 30일
  감사 로그  → loki{job="audit-logs"}   보관: 5년, 접근 제한
  시스템 로그 → loki{job="system-logs"} 보관: 90일
  보안 이벤트 → SIEM 전용 채널           보관: 5년

[RBAC 분리]
  일반 개발자: app-logs만 조회 가능
  보안팀: audit-logs + security 조회 가능
  감사팀: audit-logs 읽기 전용
```

### 4.3 로그 위·변조 방지

```
규제 요건: 로그 위·변조 방지 조치 (ISMS-P 2.9.4)

방법:
  1. 앱 → 로그 시스템 직접 쓰기 금지 (에이전트 경유만)
  2. 로그 저장소 쓰기 전용 계정 (삭제 권한 없음)
  3. WORM 스토리지 (S3 Object Lock, Glacier)
  4. 로그 체크섬 (SHA-256 해시 주기적 검증)
  5. 감사팀 전용 Kibana Role (수정·삭제 불가)
```

---

## 5. 규모별 권장 아키텍처

### 소규모 (서버 1~5대)

```
앱 → stdout → systemd journald → logrotate (90일)
        │
        └─ rsyslog → 원격 syslog 서버 (별도 장비)  ← 최소 이중화
```

### 중규모 (서버 5~50대 / K8s 소규모)

```
앱 → stdout → Alloy DaemonSet → Loki (단일 노드 또는 S3 backend)
                                    → Grafana 대시보드
감사 로그 → 별도 Loki 인스턴스 → S3 장기 보관 (ILM)
```

### 대규모 (서버 50+ / K8s 대규모 / CSAP 등급 대상)

```
앱 → stdout → Alloy → Kafka (버퍼·내구성 보장) → Logstash → Elasticsearch
                                                              └─ Kibana
감사 로그 → 별도 ES 인덱스 → S3 Object Lock (WORM, 5년)
              └─ 감사팀 전용 Kibana Space
```

---

## 6. 로그 품질 체크리스트

```
✅ 타임스탬프: UTC + 밀리초 정밀도 (서버 간 NTP 동기화 필수)
✅ 구조화 로그: JSON 포맷 (파싱 오버헤드 없음)
✅ 요청 추적: 모든 로그에 requestId / traceId 포함
✅ 레벨 사용: ERROR(즉시 대응) / WARN(주의) / INFO(운영) / DEBUG(개발)
✅ 민감 정보: 비밀번호·카드번호·주민번호 → 로그에 절대 출력 금지
✅ 성능: 로그 I/O가 요청 처리 지연 초래 금지 (비동기 appender)
✅ 중앙 전송: 서버 재시작 전 버퍼 flush 확인
✅ 감사 로그 별도: 일반 로그와 분리, 보관 기간·접근 권한 다름
```

```java
// 비동기 로그 (성능 — 로그 I/O가 요청 블로킹 방지)
// logback-spring.xml
<appender name="ASYNC_JSON" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>4096</queueSize>
    <discardingThreshold>0</discardingThreshold>   // 큐 가득차도 버리지 않음
    <appender-ref ref="JSON"/>
</appender>
```

---

## 7. 비용·샘플링·보존·PII (운영 심화) ⭐

> [!warning] 로그는 **관측성 비용의 최대 항목**이자 **PII 유출의 최대 경로**
> 메트릭은 카디널리티가 죽이고([[Metrics]] §6), 로그는 **볼륨**이 죽인다. 둘은 대칭 함정이다.

### 7.1 로그 볼륨 폭발 → 비용

```
비용 = 생성량 × (수집 + 저장 + 인덱싱 + 보존기간)
  고볼륨 INFO/DEBUG, 요청당 수십 줄, 디버그 잔존 → TB/일 → 수집·인덱싱 비용 폭증
  특히 인덱싱(Elasticsearch 등)이 비싼 항목 — "전부 인덱싱"이 사고
대응:
  - 레벨 규율: 운영 기본 INFO, DEBUG는 기본 끄고 동적 활성(§7.2)
  - 한 줄에 구조화 필드로(여러 줄 분산 금지) → 중복·반복 로그 억제
  - 인덱싱 대상 필드 한정(전문 인덱싱 ≠ 저장). 원문은 저비용 스토어(S3)로
```

### 7.2 샘플링·동적 레벨

```
샘플링: 성공·고빈도 경로는 1/N 샘플링, 에러·느린 요청은 전량
  (trace 샘플링과 연동 — 같은 요청은 로그·trace 함께 남기거나 함께 버림)
동적 레벨: 평소 INFO → 장애 시 특정 패키지만 런타임에 DEBUG로 상향
  (Spring Boot Actuator /loggers 엔드포인트 — 재배포 없이 변경)
  → 디버그 로그를 상시 켜두지 않고 "필요할 때만" → 볼륨·비용 절감
```

### 7.3 보존(retention)·계층화

```
핫(빠른 검색, 며칠) → 웜(느린 검색, 수주) → 콜드/아카이브(객체스토리지, 장기)
  최근 것만 비싼 인덱스에, 오래된 것은 저비용 보관 → ILM(Index Lifecycle Mgmt)
규제 보존기간: 감사 로그는 법·정책상 보존(§2) — 일반 로그와 보존정책 분리
  (한국 N2SF·ISMS-P 등 → [[../architecture/Public-Network-Security]])
```

### 7.4 PII·시크릿 마스킹 (컴플라이언스)

```
로그는 개인정보·시크릿이 가장 자주 새는 경로:
  - 전체 요청/응답 바디 덤프 → 주민번호·카드·토큰·비밀번호 유출
  - 예외 스택트레이스·쿼리 파라미터에 PII 섞임
대응:
  - 필드 단위 redaction(구조화 로그에서 민감 키 마스킹) — ad-hoc 정규식보다 안전
  - 마스킹은 "생성 시점"에(수집 후 지우기는 이미 늦음·복제됨)
  - 시크릿은 로깅 금지 + 스캐너로 CI에서 누출 탐지
  - DTO/엔티티 직렬화 시 @JsonIgnore·toString 마스킹 → [[DTO]]
```

### 7.5 메트릭과의 분담

```
"이건 메트릭? 로그?" — 값의 종류가 유한·집계 대상이면 메트릭, 개별 사건 상세면 로그
  고카디널리티(userId 등)는 로그/추적에, 메트릭 레이블엔 금지([[Metrics]] §6.1)
  → 메트릭으로 "무엇이/얼마나", 로그로 "왜", 추적으로 "어디서"
```

---

## 8. 관련
- [[Metrics]] — 카디널리티 폭발(메트릭측 대칭 함정, §6) · 3축 분담
- [[PLG-Stack]] · [[ELK-Stack]] — 중앙 로그 스토어 선택
- [[Prometheus-Grafana]] — 메트릭 알림과 로그 상관 분석
- [[Logging]] — Spring Boot 로그 설정
- [[../architecture/Public-Network-Security]] — N2SF·ISMS-P 감사로그 요건
