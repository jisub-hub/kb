---
tags:
  - java
  - jvm
  - tuning
  - heap
  - performance
created: 2026-06-15
---

# JVM 튜닝 — 힙 크기 · GC 옵션 · 왜 어떤 상황에서 쓰는가

> [!summary] 한 줄 요약
> JVM 튜닝의 핵심은 **왜 문제가 생겼는지 먼저 측정**하고 증상에 맞는 플래그를 적용하는 것. 측정 없는 튜닝은 오히려 악화될 수 있다.

---

## 1. 튜닝 전 반드시 측정부터

```
증상 식별 → 메트릭 수집 → 병목 확인 → 옵션 적용 → 재측정
```

**측정 도구:**
```bash
# JVM 메모리 현황
jmap -heap <pid>

# GC 통계 실시간 확인 (1초 간격)
jstat -gcutil <pid> 1000

# 힙 덤프 (OOM 분석)
jmap -dump:format=b,file=heap.hprof <pid>

# 스레드 덤프 (데드락, 블로킹 분석)
jstack <pid>

# JFR (Java Flight Recorder) — 저오버헤드 프로파일링
jcmd <pid> JFR.start duration=60s filename=recording.jfr
```

---

## 2. 힙 크기 설정

```bash
-Xms<size>    # 초기 힙 크기 (Initial heap)
-Xmx<size>    # 최대 힙 크기 (Maximum heap)
-Xss<size>    # 스레드 스택 크기 (기본 512k~1m)
```

### 왜 Xms = Xmx를 권장하는가

```bash
-Xms2g -Xmx2g    # 권장: 초기=최대, 고정 힙
```

- Xms < Xmx 면 JVM이 힙을 **동적으로 확장**하는 과정에서 Full GC 발생 가능
- 컨테이너 환경에서 힙 확장 시 OOM Killed 위험 (메모리 한계 초과)
- **고정 힙**으로 설정하면 GC 동작 예측 가능

### 컨테이너 환경 (Docker/k8s) 설정

```bash
# Java 11+ — 컨테이너 메모리 자동 감지
-XX:+UseContainerSupport          # 기본 활성화 (Java 11+)
-XX:MaxRAMPercentage=75.0         # 컨테이너 메모리의 75%를 힙으로
# (Xmx 대신 사용, 컨테이너 크기 변경에 자동 적응)
```

```yaml
# k8s 리소스 설정과 JVM 설정의 관계
resources:
  limits:
    memory: "2Gi"
# → MaxRAMPercentage=75 → -Xmx 약 1.5GB 자동 설정
# 나머지 0.5GB: Metaspace, Direct Memory, OS 오버헤드용
```

> [!warning] -Xmx를 컨테이너 limits와 같게 설정하면 OOM Killed
> JVM 힙 외에 Metaspace, Direct Buffer, JIT 컴파일 캐시 등이 추가 메모리를 사용한다. `limits`의 70~80%만 힙으로 설정.

---

## 3. Metaspace 설정

```bash
-XX:MetaspaceSize=256m       # 초기 Metaspace 크기
-XX:MaxMetaspaceSize=512m    # 최대 Metaspace (기본: 무제한 → 반드시 제한!)
```

**왜 설정해야 하는가:**
- Metaspace는 기본 **무제한**으로 OS 메모리를 계속 먹을 수 있다
- 클래스 로더 누수(ClassLoader leak)가 있으면 Metaspace가 무한 증가 → OOM
- 컨테이너 환경에서 `MaxMetaspaceSize` 미설정 → 컨테이너 메모리 초과 → OOM Killed

---

## 4. Direct Memory (Off-Heap)

```bash
-XX:MaxDirectMemorySize=512m    # NIO Direct Buffer 최대 크기
```

**언제 설정하는가:**
- Netty, WebFlux 등 **NIO 기반 프레임워크** → Direct Buffer 사용
- 기본값은 `-Xmx` 값과 동일 → 힙+Direct가 컨테이너 메모리를 초과할 수 있음
- `DirectMemoryError: Direct buffer memory` 발생 시

---

## 5. GC 관련 핵심 옵션

### G1 GC 튜닝

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200        # 목표 STW 시간 (기본 200ms)
                                 # 낮게 잡을수록 GC 더 자주 발생, 처리량 감소

-XX:G1HeapRegionSize=16m        # Region 크기 (1~32MB, 2의 거듭제곱)
                                 # 대형 객체(Humongous) 비율 높으면 크게 설정

-XX:G1NewSizePercent=20         # Young Gen 최소 비율 (기본 5%)
-XX:G1MaxNewSizePercent=40      # Young Gen 최대 비율 (기본 60%)

-XX:InitiatingHeapOccupancyPercent=45   # Old Gen이 45% 차면 Concurrent Mark 시작
                                          # 낮추면 더 일찍 GC → OOM 예방, 처리량 감소
```

**`MaxGCPauseMillis`를 언제 조정하는가:**
- 현재 200ms인데 P99 지연이 자주 200ms+ 스파이크 → `100ms`로 낮춰 더 잦지만 짧은 GC
- 배치 처리라면 `500ms`로 높여 GC 빈도 줄이고 처리량 향상

---

### ZGC 튜닝

```bash
-XX:+UseZGC
-XX:SoftMaxHeapSize=12g         # Soft 힙 한계 (ZGC가 이 값 이하 유지 목표)
-XX:ZCollectionInterval=5       # 최소 5초마다 GC 실행 (메모리 여유 있어도)
-XX:ZUncommitDelay=300          # GC 후 메모리 반환 지연 시간(초) — 컨테이너에서 중요
```

---

## 6. JIT 컴파일러 관련

```bash
-XX:+TieredCompilation          # 기본 활성화. 인터프리터 → C1 → C2 단계적 컴파일
-XX:CompileThreshold=10000      # 메서드 호출 횟수 임계값 (기본 10000)
                                 # 낮추면 빠르게 JIT, 높이면 더 많은 데이터로 최적화

# 워밍업 없는 일회성 스크립트엔 JIT 끄기 (오버헤드 줄임)
-Xint                           # 인터프리터 전용 (프로덕션 서버엔 금지)
```

---

## 7. OOM 발생 시 자동 힙 덤프

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof

# OOM 후 프로세스 자동 재시작 방지 (k8s는 알아서 재시작)
-XX:+ExitOnOutOfMemoryError
```

---

## 8. 증상별 튜닝 가이드

| 증상 | 원인 | 조치 |
|---|---|---|
| Full GC 자주 발생, STW 길다 | Old Gen 부족 | `-Xmx` 증가 또는 G1→ZGC 전환 |
| P99 응답 200ms+ 스파이크 | GC STW | `MaxGCPauseMillis` 낮추거나 ZGC 전환 |
| `OutOfMemoryError: Java heap space` | 힙 부족 또는 메모리 누수 | `-Xmx` 증가 + 힙 덤프 분석 |
| `OutOfMemoryError: Metaspace` | 클래스 로더 누수 | `MaxMetaspaceSize` 설정 + 누수 분석 |
| `OutOfMemoryError: Direct buffer memory` | NIO Direct Buffer 부족 | `MaxDirectMemorySize` 증가 |
| 컨테이너 OOM Killed (힙은 정상) | Non-heap 메모리 초과 | Metaspace/Direct 제한, `MaxRAMPercentage` 조정 |
| CPU 100% (GC 원인) | GC 스레드 과부하 | 힙 크기 증가, GC 알고리즘 변경 |
| 시작 속도 느림 | JIT 워밍업 | AOT 컴파일 (GraalVM native), AppCDS |

---

## 9. 컨테이너 최적화 JVM 플래그 모음

```bash
# Spring Boot 컨테이너 권장 설정 (Java 21, k8s)
JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:MaxMetaspaceSize=256m \
  -XX:MaxDirectMemorySize=256m \
  -XX:+UseZGC \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/heapdump.hprof \
  -Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=3,filesize=10m \
  -Djava.security.egd=file:/dev/./urandom"
```

---

## 10. 관련
- [[GarbageCollector]] · [[../testing/Load-Testing]]
- [[Resource-Management]] — K8s에서 힙·CPU 인식(컨테이너 limit↔MaxRAMPercentage·throttling·OOMKill)
