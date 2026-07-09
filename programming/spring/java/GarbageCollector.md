---
tags:
  - java
  - jvm
  - gc
  - garbage-collector
  - performance
created: 2026-06-15
---

# Garbage Collector — 종류 · 선택 기준 · 전환 시기

> [!summary] 한 줄 요약
> JVM GC는 **자동 메모리 관리** 담당. GC 종류마다 처리량(Throughput) vs 지연(Latency) 트레이드오프가 다르다. 애플리케이션 특성(배치 vs 실시간 서비스)에 따라 GC를 선택해야 한다.

---

## 1. GC가 왜 중요한가

- GC는 **Stop-The-World(STW)** 이벤트를 발생시킨다 → 모든 앱 스레드 일시 정지
- 긴 STW = 응답 지연 스파이크 → SLA 위반, 타임아웃
- 잘못된 GC 선택 또는 힙 크기 설정 → OutOfMemoryError, GC 지옥

```
[Young Gen]  →  Minor GC  (빈번, 짧음)
[Old Gen]    →  Major/Full GC  (드묾, 길고 비쌈)
```

---

## 2. GC 종류별 특성

### Serial GC
```bash
-XX:+UseSerialGC
```
- **단일 스레드** GC. 모든 GC 작업을 하나의 스레드가 처리
- STW 시간 가장 김
- 용도: **힙이 작은(<256MB) 단일 CPU 환경**, 마이크로컨테이너
- 프로덕션 서비스에는 부적합

---

### Parallel GC (Throughput GC)
```bash
-XX:+UseParallelGC          # Java 8 이전 기본값
-XX:ParallelGCThreads=4
```
- **멀티스레드**로 GC 작업 병렬 처리
- STW는 발생하지만 시간 단축
- **처리량 최대화** 목표 → STW를 감수하고 많은 데이터 처리
- 용도: **배치 처리, 빅데이터, 야간 배치 잡** → 지연보다 처리량이 중요한 경우

---

### G1 GC (Garbage First)
```bash
-XX:+UseG1GC               # Java 9+ 기본값
-XX:MaxGCPauseMillis=200   # 목표 최대 STW 시간 (기본 200ms)
```
- 힙을 **Region(1~32MB)**으로 분할, 가득 찬 Region 우선 수집
- **예측 가능한 STW 시간** 목표 (`MaxGCPauseMillis` 설정)
- Incremental compaction → 조각화 방지
- 용도: **Java 9+ 일반 서비스, 힙 6GB 이하의 대부분의 Spring Boot 앱**
- 특징: 지연과 처리량의 균형. 대부분의 웹 애플리케이션에 충분

```
힙 구성: Eden Region | Survivor Region | Old Region | Humongous Region(대형 객체)
```

---

### ZGC (Z Garbage Collector)
```bash
-XX:+UseZGC                # Java 15+ 프로덕션 권장
-Xmx16g                    # 대용량 힙에서 강점
```
- **STW ≈ 1ms** 목표 (힙 크기에 무관) → 초저지연
- 대부분의 GC 작업을 **앱 스레드와 동시(concurrent)** 실행
- 힙을 ZPage로 관리, colored pointers + load barriers 사용
- 메모리 오버헤드 약 10~20% (colored pointer 정보 저장)
- 용도:
  - **대용량 힙 (4GB~수 TB)** 서비스
  - **P99 지연 SLA가 엄격한 실시간 서비스** (금융 트랜잭션, 게임 서버)
  - 힙 크기가 자주 바뀌는 컨테이너 환경

---

### Shenandoah GC
```bash
-XX:+UseShenandoahGC       # RedHat OpenJDK, Java 15+ Oracle JDK
```
- ZGC와 유사하게 **저지연** 목표, STW ≈ 수 ms
- ZGC보다 **낮은 메모리 오버헤드**, 소형 힙에서도 효과적
- 용도: ZGC 대안, Red Hat 기반 환경

---

## 3. GC 비교표

| GC | STW 시간 | 처리량 | 힙 크기 | Java 버전 | 적합한 상황 |
|---|---|---|---|---|---|
| **Serial** | 길다 | 낮음 | 소형 (<256MB) | 모든 버전 | 임베디드, 마이크로서비스 |
| **Parallel** | 중간 | **최고** | 중형 | 모든 버전 | 배치 처리 |
| **G1** | 짧음 (예측 가능) | 높음 | 중대형 (≤6GB) | **9+ 기본** | 대부분의 Spring Boot 앱 |
| **ZGC** | **≈1ms** | 중간 | 대형 (4GB~TB) | 15+ | 초저지연 필요 |
| **Shenandoah** | **≈수ms** | 중간 | 중대형 | 15+ | ZGC 대안 |

---

## 4. GC 전환 시점

### Parallel → G1로 전환
- Java 8 이하 → **Java 11/17/21 업그레이드 시** (G1이 기본)
- Full GC가 자주 발생하고 STW가 길어질 때
- 힙 크기가 2GB 이상으로 커질 때

### G1 → ZGC로 전환
- P99 응답 지연이 200ms 이상 → GC STW가 원인으로 확인될 때
- 힙이 4GB 이상이고 지연 SLA가 엄격할 때 (예: 100ms 이하 P99)
- **GC 로그 분석 후 STW가 병목**임을 확인한 다음 전환

> [!warning] 성능 측정 없이 무작정 ZGC로 전환하지 마라
> ZGC는 처리량이 G1보다 낮을 수 있다. 배치·배경 작업에는 G1/Parallel이 더 나을 수 있다.

---

## 5. GC 로그 분석

```bash
# GC 로그 활성화 (Java 9+)
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=20m

# 주요 확인 항목
[GC pause (G1 Young Generation) ... 15.123ms]  ← STW 시간
[Full GC (Ergonomics) ... 1.234s]               ← Full GC (위험 신호)
```

**분석 도구:**
- **GCEasy** (gcease.io) — GC 로그 웹 업로드 분석
- **GCViewer** — 오픈소스 GUI 분석기
- **JVM Mon** — 실시간 JVM 모니터링

---

## 6. 관련
- [[JVM-Tuning]] · [[../testing/Load-Testing]]
