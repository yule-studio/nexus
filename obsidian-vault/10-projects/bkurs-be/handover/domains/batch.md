---
title: "도메인 — batch (Spring Batch / Cron)"
kind: handover
status: stable
created_at: 2026-05-22T00:08:00+09:00
tags: [bkurs-be, handover, domain, batch, cron]
related:
  - "[[../../bkurs-be]]"
  - "[[../async-mq-batch]]"
home_hub: "[[../../bkurs-be]]"
---

# batch 도메인

> Spring Batch 기반 cron job. 현재 1개 — `overduePaymentAlertJob` (미납 학원 daily 알림). job
> 자동 실행 토글 `bkurs.batch.scheduling.enabled` (default true).

## 1. 책임

- daily / weekly 정기 작업
- Spring Batch metadata 테이블 활용 (V24)
- 작업 결과를 `MessagingPublisher` 로 publish (consumer 는 별도)

## 2. 핵심 파일

```
batch/
├── OverduePaymentAlertJobConfig.java   # Job + Step + Reader/Processor/Writer
└── ScheduledBatchLauncher.java         # @Scheduled cron → JobLauncher.run(...)
```

## 3. OverduePaymentAlertJob

목적: 결제 미납 학원을 학원장에게 daily 03:00 알림.

```java
@Configuration
public class OverduePaymentAlertJobConfig {

    @Bean
    public Job overduePaymentAlertJob(JobRepository jobRepo, Step step) {
        return new JobBuilder("overduePaymentAlertJob", jobRepo).start(step).build();
    }

    @Bean
    public Step overduePaymentAlertStep(JobRepository jobRepo, PlatformTransactionManager tm,
                                         ItemReader<Academy> reader,
                                         ItemProcessor<Academy, NotificationRequest> processor,
                                         ItemWriter<NotificationRequest> writer) {
        return new StepBuilder("overduePaymentAlertStep", jobRepo)
            .<Academy, NotificationRequest>chunk(10, tm)
            .reader(reader).processor(processor).writer(writer).build();
    }

    // reader: 미납 학원 조회
    // processor: NotificationRequest 변환
    // writer: messagingPublisher.publish(PAYMENT_OVERDUE, payload) (또는 notificationService.send 직접)
}
```

## 4. ScheduledBatchLauncher

```java
@Component
@ConditionalOnProperty(name = "bkurs.batch.scheduling.enabled",
                       havingValue = "true", matchIfMissing = true)
@RequiredArgsConstructor
@Slf4j
public class ScheduledBatchLauncher {

    private final JobLauncher jobLauncher;
    private final @Qualifier("overduePaymentAlertJob") Job overduePaymentAlertJob;

    @Scheduled(cron = "0 0 3 * * *", zone = "Asia/Seoul")
    public void runOverduePayment() throws Exception {
        JobParameters params = new JobParametersBuilder()
            .addLong("ts", System.currentTimeMillis())
            .toJobParameters();
        jobLauncher.run(overduePaymentAlertJob, params);
    }
}
```

`@EnableScheduling` 은 `common/config/AsyncConfig` 에서 한다. test 환경에서는
`bkurs.batch.scheduling.enabled=false` 로 자동 실행 회피.

## 5. Spring Batch metadata (V24)

`BATCH_JOB_INSTANCE / BATCH_JOB_EXECUTION / BATCH_STEP_EXECUTION ...` — Spring Batch 가 관리.
중복 실행 / 재개 / 실패 추적에 사용.

application yml:

```yaml
spring:
  batch:
    job:
      enabled: false   # startup 자동 실행 비활성 — ScheduledBatchLauncher 가 trigger
```

## 6. 의존

- **의존**: academy (미납 학원), plan.payment, notification (메시지 전달), messaging
- **의존받음**: (직접 없음 — 단독 cron)

## cross-link

- [[../async-mq-batch]] — async / mq / batch SSoT
- [[plan]]
- [[notification]]
