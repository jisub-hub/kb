---
tags:
  - spring
  - batch
  - scheduling
  - etl
  - quartz
created: 2026-06-17
---

# Spring Batch & 스케줄링

> [!summary] 한 줄 요약
> **Spring Batch**는 대용량 데이터를 **Job → Step → Chunk(ItemReader→Processor→Writer)** 구조로 처리하는 배치 프레임워크. **JobRepository**로 실행 메타데이터를 관리해 **재시작·skip·retry**가 가능. 단순 주기 실행은 `@Scheduled`, 복잡한 스케줄은 Quartz, 대량 데이터 처리는 Spring Batch로 역할을 나눈다.

---

## 1. 배치가 필요한 경우
실시간 요청-응답으로 처리하기엔 무겁거나, 일괄/주기 처리가 자연스러운 작업.

- **ETL**: 외부 데이터를 추출·변환·적재 (DB → DW, 로그 집계)
- **정산/집계**: 일/월 마감, 수수료 계산, 리포트 생성
- **데이터 정리**: 만료 데이터 삭제, 아카이빙, 인덱스 재구성
- **재처리(reprocessing)**: 실패 건 재시도, 누락 데이터 보정
- **대량 통지**: 푸시/메일 발송 큐 비우기

실시간이 아닌 **처리량(throughput)** 이 핵심이며, 멱등성과 재시작 가능성이 중요하다.

---

## 2. Spring Batch 구조

```
Job (배치 작업 단위)
 └── Step (단계, 순차/조건 실행)
      └── Chunk 지향 처리
           ItemReader  → ItemProcessor → ItemWriter
           (한 건 읽기)   (변환/필터)      (chunk 단위 일괄 쓰기)

           [ read read read ... (chunk-size 만큼) → process → write → commit ]
```

- **Job**: 하나의 배치 실행 단위. 여러 Step을 가질 수 있다.
- **Step**: Chunk 지향 또는 Tasklet(단일 작업) 방식.
- **Chunk 지향**: `chunk-size` 만큼 read/process 후 한 번에 write + 트랜잭션 커밋.

```java
@Configuration
@RequiredArgsConstructor
public class SettlementJobConfig {

    @Bean
    Job settlementJob(JobRepository repo, Step settlementStep) {
        return new JobBuilder("settlementJob", repo)
                .start(settlementStep)
                .build();
    }

    @Bean
    Step settlementStep(JobRepository repo, PlatformTransactionManager tx,
                        ItemReader<Order> reader,
                        ItemProcessor<Order, Settlement> processor,
                        ItemWriter<Settlement> writer) {
        return new StepBuilder("settlementStep", repo)
                .<Order, Settlement>chunk(500, tx)   // chunk = 커밋 단위
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .faultTolerant()
                .skip(ParseException.class).skipLimit(10)
                .retry(DeadlockLoserDataAccessException.class).retryLimit(3)
                .build();
    }
}
```

---

## 3. Chunk 커밋 단위
`chunk-size`는 **성능과 안정성의 트레이드오프**.

| chunk-size | 장점 | 단점 |
|------------|------|------|
| 크게 (예: 1000+) | 커밋 횟수 감소 → 처리량 ↑ | 메모리 ↑, 롤백 시 손실 범위 ↑ |
| 작게 (예: 50) | 롤백 영향 최소, 메모리 ↓ | 커밋 오버헤드 ↑ |

- 한 chunk 처리 중 예외 발생 시 **해당 chunk 전체 롤백** → 재시작은 마지막 커밋 지점부터.
- I/O 비용·트랜잭션 크기·메모리를 고려해 보통 100~1000 사이에서 튜닝.

---

## 4. JobRepository (메타데이터)
Spring Batch는 실행 이력을 메타 테이블(`BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 등)에 기록한다.

- **JobInstance**: Job 이름 + JobParameters 조합. 동일 파라미터로는 중복 실행 불가(완료 시).
- **JobExecution / StepExecution**: 실행 상태(STARTED/COMPLETED/FAILED), read/write/skip 카운트, 커밋 지점 저장.
- 이 메타데이터 덕분에 **재시작 시 이어서 처리**가 가능하다.

```sql
-- 진행 상황 / 실패 추적
SELECT step_name, status, read_count, write_count, commit_count, rollback_count
FROM BATCH_STEP_EXECUTION
WHERE job_execution_id = ?;
```

---

## 5. Fault Tolerance — 재시작 / skip / retry

- **재시작(restart)**: FAILED Job을 동일 JobParameters로 재실행하면 마지막 커밋된 chunk 다음부터 처리.
- **skip**: 특정 예외(데이터 오류 등)는 건너뛰고 진행. `skipLimit` 초과 시 Job 실패.
- **retry**: 일시적 예외(데드락, 네트워크)는 지정 횟수 재시도.
- **listener**로 skip/retry/실패 건을 별도 테이블·로그·[[Kafka]] 토픽에 적재해 추후 재처리.

```java
.faultTolerant()
.skip(FlatFileParseException.class).skipLimit(100)
.retry(TransientDataAccessException.class).retryLimit(3)
.listener(new SkipListener<Order, Settlement>() {
    @Override public void onSkipInProcess(Order item, Throwable t) {
        deadLetterRepository.save(item, t);   // 실패 건 별도 보관 → 재처리
    }
});
```

---

## 6. 파티셔닝 · 병렬 Step
처리량을 높이는 병렬화 전략.

| 방식 | 설명 | 용도 |
|------|------|------|
| Multi-threaded Step | 한 Step 내 chunk를 멀티스레드로 | 간단한 병렬, reader thread-safe 필요 |
| Parallel Steps | 독립 Step을 동시 실행 | 의존 없는 작업 병렬 |
| **Partitioning** | 데이터를 파티션으로 분할해 worker가 분담 | 대용량, 분산 처리(원격 파티셔닝) |
| Remote Chunking | reader는 마스터, processing/write는 워커 | 처리 비용이 큰 경우 |

```java
@Bean
Step masterStep(JobRepository repo, Step workerStep, Partitioner partitioner) {
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", partitioner)   // 예: ID 범위로 분할
            .step(workerStep)
            .gridSize(8)                               // 파티션 수
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .build();
}
```

---

## 7. 스케줄링 비교 — @Scheduled vs Quartz vs Spring Batch
세 가지는 경쟁재가 아니라 **역할이 다르다** (조합해서 사용).

| 구분 | @Scheduled | Quartz | Spring Batch |
|------|-----------|--------|--------------|
| 역할 | 단순 주기 실행 | 복잡한 스케줄 + 영속 | 대량 데이터 처리 엔진 |
| 트리거 | 고정 cron/fixedRate | cron, calendar, 동적 | 직접 트리거(스케줄러 필요) |
| 상태 영속 | ❌ (메모리) | ✅ (JobStore/DB) | ✅ (JobRepository) |
| 분산/클러스터 | ❌ | ✅ (클러스터 모드) | 메타DB 공유 |
| 재시작/재처리 | ❌ | 제한적 | ✅ (핵심 기능) |
| 적합 | 캐시 갱신, 헬스체크 | 운영자 변경 가능한 스케줄 | ETL, 정산, 대량 처리 |

> 실무 패턴: **Quartz/`@Scheduled`가 Spring Batch Job을 트리거** → 스케줄링과 처리를 분리.

```java
@Scheduled(cron = "0 0 1 * * *")   // 매일 01:00
public void runSettlement() {
    jobLauncher.run(settlementJob,
        new JobParametersBuilder()
            .addLocalDate("runDate", LocalDate.now().minusDays(1))  // 멱등 키
            .toJobParameters());
}
```

---

## 8. 분산 환경 중복 실행 방지
스케줄러가 N개 인스턴스에서 동시에 뜨면 **같은 Job이 중복 실행**된다. 방지 수단:

- **ShedLock**: 가장 가벼운 해법. DB/Redis에 락 레코드를 두고 한 인스턴스만 실행.
- **분산 락**: Redis(Redisson)/ZooKeeper 기반 → [[Distributed-Lock]] 참고.
- **Quartz 클러스터 모드**: 동일 DB JobStore를 공유해 한 노드만 트리거.

```java
@Scheduled(cron = "0 0 1 * * *")
@SchedulerLock(name = "settlementJob",
               lockAtLeastFor = "PT5M", lockAtMostFor = "PT55M")
public void runSettlement() { /* ShedLock이 단일 실행 보장 */ }
```

JobParameters를 멱등 키(날짜 등)로 두면, 중복 트리거되어도 동일 JobInstance는 재실행되지 않아 이중 안전망이 된다. 다운스트림은 [[Eventual-Consistency]]를 전제로 설계한다.

---

## 9. 모니터링
- **메타 테이블 직접 조회**: 실패/지연 Job 추적.
- **Spring Batch Actuator/Micrometer**: `spring.batch.*` 메트릭을 Prometheus로 노출(처리 건수, 실패율, 소요시간).
- **알림**: Job 실패 시 listener에서 Slack/메일/PagerDuty 발송.
- **대시보드**: chunk 처리 속도, skip/retry 추이, 마지막 성공 시각을 그래프화.

```java
public class JobMonitoringListener implements JobExecutionListener {
    @Override public void afterJob(JobExecution exec) {
        if (exec.getStatus() == BatchStatus.FAILED) {
            alertService.notifyFailure(exec.getJobInstance().getJobName(),
                                       exec.getAllFailureExceptions());
        }
    }
}
```

---

## 10. 관련
- [[Distributed-Lock]] · [[Kafka]] · [[Eventual-Consistency]]
