---
title: "도메인 — notification (Email / SMS Gateway)"
kind: handover
status: stable
created_at: 2026-05-22T00:05:00+09:00
tags: [bkurs-be, handover, domain, gateway, notification, sendgrid]
related:
  - "[[../../bkurs-be]]"
  - "[[../external-gateways]]"
home_hub: "[[../../bkurs-be]]"
---

# notification 도메인

> Email / SMS / Push 알림 gateway. Mock ↔ SendGrid (PR #102 사이클 2) swap. 비동기 dispatcher 가
> `externalCallExecutor` 사용. controller 없음 — 다른 도메인 service 가 호출.

## 1. 책임

- `NotificationService.send(NotificationRequest)` — 동기
- `AsyncNotificationDispatcher.dispatch(...)` — 비동기 (`@Async("externalCallExecutor")`)
- 실 구현 SendGrid (email) — SMS / Push 는 후속

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `NotificationService` (interface) | `send(request) → NotificationResult` |
| `NotificationRequest` (record) | channel / to / subject / body / data map |
| `NotificationChannel` (enum) | EMAIL / SMS / PUSH |
| `NotificationResult` (record) | success / providerMessageId / error |
| `AsyncNotificationDispatcher` | `@Async` wrapper — caller 가 fire-and-forget |
| `MockNotificationService` | `@Profile("!prod")` — log 만 |
| `SendGridNotificationService` | `@Profile("prod")` — SendGrid SDK |
| `SendGridProperties` | `@ConfigurationProperties("sendgrid.*")` |

## 3. 패키지 구성

```
notification/
└── gateway/
    ├── NotificationService              # interface
    ├── NotificationRequest               # record
    ├── NotificationResult                # record
    ├── NotificationChannel               # enum
    ├── MockNotificationService          # @Profile("!prod")
    ├── SendGridNotificationService      # @Profile("prod")
    ├── SendGridProperties               # @ConfigurationProperties
    └── AsyncNotificationDispatcher      # @Async dispatcher
```

> controller 없음 — 다른 도메인 service 가 caller.

## 4. caller 도메인

| caller | 호출 시점 |
| --- | --- |
| `academy.TeacherInviteService` | invite 발급 → 강사 이메일 자동 발송 |
| `academy.OwnerTransferService` | 이관 신청 → 수신자에게 알림 |
| `batch.OverduePaymentAlertJobConfig` | 미납 학원 → 학원장 알림 |
| `plan.SubscriptionService` | (후속) 구독 갱신 / 결제 실패 |

## 5. 핵심 코드 흐름

```java
@Service
@RequiredArgsConstructor
public class TeacherInviteService {
    private final AsyncNotificationDispatcher dispatcher;

    public TeacherInvite invite(String teacherEmail, long academyId) {
        TeacherInvite invite = TeacherInvite.create(...);
        teacherInviteRepository.save(invite);

        // 비동기 이메일 — invite save 후 발송, 실패해도 invite 는 commit
        dispatcher.dispatch(NotificationRequest.email(
            teacherEmail,
            "[B-KURS] 강사 초대",
            renderInviteBody(invite),
            Map.of("inviteToken", invite.getToken())));

        return invite;
    }
}
```

## 6. 비동기 실행

`AsyncNotificationDispatcher.dispatch` 는 `@Async("externalCallExecutor")` — saturation 격리.
호출자 트랜잭션과 무관 (별 스레드). 실패 시 log only — 현재 retry 없음.

후속:
- retry (Resilience4j)
- dead letter queue (Rabbit)
- batch send (SendGrid batch endpoint)

## 7. 의존

- **의존**: common.config.AsyncConfig
- **의존받음**: academy, batch, plan (후속)

## cross-link

- [[../external-gateways#notification]]
- [[batch]]
- [[../async-mq-batch]]
