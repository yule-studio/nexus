---
title: "bkurs-be — 외부 SaaS Gateway 인수인계"
kind: handover
status: stable
created_at: 2026-05-21T22:30:00+09:00
tags: [bkurs-be, handover, gateway, saas]
related:
  - "[[../bkurs-be]]"
  - "[[overview]]"
  - "[[hybrid-external-services]]"
  - "[[services-guide]]"
  - "[[services-accounts]]"
home_hub: bkurs-be
---

# bkurs-be — 외부 SaaS Gateway 인수인계

> **모든 외부 의존은 interface + Mock + 실 구현** 패턴. local/test 는 Mock, prod 는 실 SaaS.
> 발급/설정 가이드는 [[services-guide]], 자격 정보 inventory 는
> [[services-accounts]] (gitignored). 본 문서는 **코드 SSoT**.

## 1. 공통 패턴

모든 gateway 도메인이 동일.

```
{domain}/gateway/
├── {Foo}Service.java          # interface — caller 가 이것만 의존
├── Mock{Foo}Service.java      # @Profile("!prod") — local + test default
├── {Real}{Foo}Service.java    # @Profile("prod") — 실 SaaS
└── {Foo}Properties.java       # @ConfigurationProperties(prefix="...") — API key / endpoint
```

호출 측 (도메인 service):

```java
@Service
@RequiredArgsConstructor
public class TeacherInviteService {
    private final NotificationService notificationService;   // ← interface 의존

    public void invite(...) {
        // ...
        notificationService.sendAsync(NotificationRequest.email(...));
    }
}
```

비동기: `@Async("externalCallExecutor")` 또는 호출 측 service 가 직접 executor 사용. saturation
isolation 은 `AsyncConfig` 2 pool 분리 ([[common#7.3]]).

## 2. file — `com.bkurs.file.gateway`

| 항목 | 값 |
| --- | --- |
| interface | `FileStorageService` |
| Mock | `MockFileStorage` (local FS / in-memory URL) |
| 실 구현 | (사이클 2 후속 — S3 / Supabase Storage / GCS) |
| props | (필요 시 `FileStorageProperties` 신설) |
| caller | `interview` (audio/video), `pdf` (학생부 출력), `record` (첨부) |

핵심 메서드:

```java
UploadUrlIssuance issuePresignedUploadUrl(String key, Duration ttl);
```

응답 `UploadUrlIssuance` = `{ url, fields }` — 클라이언트가 직접 PUT 으로 업로드 (서버는 url 만
발급, 파일은 거치지 않음). 자세히 [[envelope-migration]] B17 항목.

## 3. notification — `com.bkurs.notification.gateway`

| 항목 | 값 |
| --- | --- |
| interface | `NotificationService` |
| Mock | `MockNotificationService` (log 만) |
| 실 구현 | `SendGridNotificationService` (PR #102 사이클 2) |
| props | `SendGridProperties` (`sendgrid.api-key`, `sendgrid.from-email`) |
| async dispatcher | `AsyncNotificationDispatcher` |

타입:

```java
public record NotificationRequest(
    NotificationChannel channel,    // EMAIL / SMS / PUSH
    String to,
    String subject,
    String body,
    Map<String, Object> data
) { ... }

public record NotificationResult(boolean success, String providerMessageId, String error) { ... }
```

호출:

```java
notificationService.send(NotificationRequest.email(to, subject, body, data));
// 또는 비동기:
asyncDispatcher.dispatch(request);
```

caller: `TeacherInviteService` (invite email), `batch.OverduePaymentAlertJobConfig` (미납 알림),
`AcademyOwnerTransferService` (이관 confirm 알림).

## 4. pdf — `com.bkurs.pdf.gateway`

| 항목 | 값 |
| --- | --- |
| interface | `PdfGenerator` |
| Mock | `MockPdfGenerator` (text 만 byte[] 로) |
| 실 구현 | (후속 — Puppeteer microservice or AWS Lambda chromium) |
| controller | `StudentRecordPdfController` (3 endpoint — student/teacher/owner) |

핵심:

```java
byte[] generate(String html);              // HTML → PDF
byte[] generateFromTemplate(String tplKey, Map<String, Object> vars);
```

caller: `pdf` 컨트롤러 (학생부 출력). 향후 ai-summary 모듈 등이 호출 가능.

## 5. ai — `com.bkurs.ai.gateway`

| 항목 | 값 |
| --- | --- |
| interface | `AiAssistant` |
| Mock | (생략 — 또는 echo) |
| 실 구현 | `ClaudeAiAssistant` (PR #103 사이클 2) |
| props | `claude.api-key`, `claude.model` (claude-sonnet-4-6 default 권장) |
| 호출 record | `AiAnalysisRequest` (system + user prompt + max_tokens) |

caller: 후속 사이클 — 자소서 첨삭 / 면접 답변 분석 / 토론 평가 보조.

> ⚠️ Anthropic SDK 사용 시 **prompt caching** 활성화 (system prompt 캐싱). model id 는
> `claude-sonnet-4-6` 또는 `claude-opus-4-7` 사용. retired model 회피.

## 6. stt — `com.bkurs.stt.gateway`

| 항목 | 값 |
| --- | --- |
| interface | (Clova STT 추상) |
| Mock | (PR #104 사이클 2) |
| 실 구현 | Clova STT REST adapter |
| props | `clova.client-id`, `clova.client-secret`, `clova.endpoint` |

caller: `interview.InterviewResponseService` (audio URL → 텍스트 transcript).

## 7. payment — `com.bkurs.plan.gateway`

| 항목 | 값 |
| --- | --- |
| interface | `PaymentGateway` |
| Mock | `MockPaymentGateway` (instant SUCCESS / 일정 비율 FAILED) |
| 실 구현 | `PortOnePaymentGateway` (PR #101 사이클 2, KR 통합 결제) |
| props | `PortOneProperties` (`portone.imp-key`, `portone.imp-secret`, `portone.merchant-id`) |
| caller | `PaymentService`, `SubscriptionService.chargeOrFail / cancelAndRefund` |

핵심:

```java
PaymentResult charge(ChargeRequest req);    // PG 호출, idempotent key 사용
PaymentResult refund(String pgTxnId, Money amount);
```

webhook 처리: 별도 endpoint (PortOne → POST /webhook/portone) — signature 검증 후 `Payment` row
업데이트. 후속 사이클 항목.

## 8. profile 전환

| 환경 | active profile | 구현 |
| --- | --- | --- |
| local | `local` | Mock 모두 |
| dev | `dev` | Mock 또는 sandbox SaaS |
| prod | `prod` | 모두 실 SaaS |

전환 키: `SPRING_PROFILES_ACTIVE=prod`. 각 `@Component` 는 `@Profile("prod")` / `@Profile("!prod")`.

## 9. 새 외부 SaaS 추가 절차

1. 새 도메인 패키지: `com.bkurs.{newgw}/gateway/`.
2. interface 작성 — caller 가 의존할 메서드만 명시. Mock 부터 작성.
3. `@ConfigurationProperties` properties record — `application-prod.yml` 의 env 키 매핑.
4. Mock 구현 — `@Profile("!prod")`. log + 합리적 fake response.
5. 실 구현 — `@Profile("prod")`. SDK / REST adapter.
6. 호출 측 service 에서 `@Async("externalCallExecutor")` 로 비동기. saturation isolation.
7. CI / 통합 테스트는 Mock 으로 GREEN. 실 호출 smoke 는 dev 환경 한정 수동.
8. 자격 정보는 절대 commit 금지 — `.env.example` 키만 추가, 실제 값은 secret manager.

## 10. fallback / 회로 차단 (현재 미구현, 후속)

- timeout / retry / circuit breaker 는 Resilience4j 도입 검토. 현재는 SDK 기본 timeout 만.
- queueing: Rabbit (`MessagingPublisher`) 로 외부 호출 큐잉 가능. consumer worker 는 후속.

## cross-link

- [[../bkurs-be]]
- [[overview]] — async / messaging / batch
- [[hybrid-external-services]] — 외부 SaaS hybrid 전략 SSoT
- [[services-guide]] — SaaS 발급 / 설정
- [[services-accounts]] — 자격 inventory (gitignored)
- [[envelope-migration]] — 도메인 별 envelope migration 상태
