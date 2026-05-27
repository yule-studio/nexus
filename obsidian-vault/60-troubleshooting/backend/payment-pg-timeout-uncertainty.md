---
title: "troubleshooting · PG 응답 timeout — 결제 상태 불명"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, payment, timeout, reconciliation]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[payment-double-charge]]"
  - "[[payment-webhook-idempotency]]"
---

# PG 응답 timeout — 결제됐는지 모름

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — PENDING / reconciliation / status query |

## 증상

```
java.net.SocketTimeoutException: Read timed out
    at com.example.TossPaymentClient.charge(TossPaymentClient.java:42)
```

상황:
- 서버 → PG `charge()` 호출 → 5초 후 응답 없음 → timeout 예외
- **사용자 카드는 charge 됐을 수도, 안 됐을 수도** — 응답 못 받음
- 서버가 FAIL 처리 + 사용자에게 "결제 실패" 응답 → 사용자가 다시 시도 → **이미 됐으면 2회 결제**, 아니면 정상
- 카드 명세서에 1건만 / 2건 모두 가능

## 가설

- timeout 됐으면 PG 가 charge 안 한 거 아닌가? → 틀림. PG 도 처리는 끝났는데 응답이 안 옴 (network 만 끊김)
- 그냥 retry 하면? → 같은 거래의 중복 charge 위험
- 사용자에게 "확인 중" 안내? → UX 나쁨 / 자금 흐름 불명

실제: **분산 시스템의 partial failure** — 양쪽 다 모르는 상태를 명시적으로 reconcile 해야 함.

## 원인 (root cause)

HTTP request/response 는 양방향. response 미수신 = 3가지 가능:

```
A. PG 가 받지 못함 → charge 안 됨 (재시도 안전)
B. PG 가 처리 중 → charge 됐을 수도 / 안 됐을 수도
C. PG 가 완료했는데 응답만 손실 → charge 됨 (재시도 = 중복)
```

client (server) 가 어느 case 인지 알 수 없음. **단순 retry 는 case C 에서 double charge**.

## 해결

### 원칙

**(1) DB 에 `PENDING` 상태로 선기록** — 결과 모름을 명시
**(2) 사용자에게 즉시 "확인 중" 표시** — FAIL 로 단정 X
**(3) PG status query API + reconciliation 으로 진실 추적**
**(4) Webhook 도 신호** — 양쪽 모두 처리, 먼저 도착한 쪽이 winner

### 구현

**(1) Timeout 명시 — 무한 대기 차단**

```java
@Bean
public RestClient pgRestClient() {
    var f = new HttpComponentsClientHttpRequestFactory();
    f.setConnectTimeout(2000);   // 2초 — connect 안 되면 case A
    f.setReadTimeout(5000);      // 5초 — 응답 안 오면 case B/C 가능
    return RestClient.builder().requestFactory(f).build();
}
```

**(2) PENDING + idempotency-key 패턴** — [[payment-double-charge]] 참조

```java
@Service
public class PaymentService {

    @Transactional
    public PaymentResponse process(String idempotencyKey, PaymentRequest req) {
        Payment p = paymentRepo.findByIdempotencyKey(idempotencyKey)
            .orElseGet(() -> paymentRepo.save(Payment.pending(idempotencyKey, req)));

        if (p.isTerminal()) return PaymentResponse.from(p);   // 이미 PAID/FAILED

        try {
            var pg = pgClient.chargeWithIdempotency(idempotencyKey, req);   // PG 도 idempotent
            p.markPaid(pg.transactionId());
        } catch (PgTimeoutException timeout) {
            // 중요: FAIL 로 단정 X — PENDING 유지
            log.warn("PG timeout, will reconcile: key={}", idempotencyKey);
            // p.status = PENDING (그대로)
        } catch (PgException definitelyFailed) {
            // PG 가 명시적 FAIL 응답 (4xx with error code)
            p.markFailed(definitelyFailed.getCode());
        }
        paymentRepo.save(p);
        return PaymentResponse.from(p);
    }
}
```

응답:
```json
{
  "paymentId": 123,
  "status": "PENDING",
  "message": "결제 확인 중입니다. 잠시 후 다시 확인해주세요."
}
```

→ frontend 가 polling 또는 webhook event 로 최종 결과 받음.

**(3) PG status query API 호출** — 진실 추적

```java
@Component
@RequiredArgsConstructor
public class PaymentReconciler {
    private final PaymentRepository paymentRepo;
    private final PgClient pgClient;

    @Scheduled(fixedDelay = 60_000)   // 1분 마다
    public void reconcilePending() {
        var stale = paymentRepo.findByStatusAndCreatedAtBefore(
            PaymentStatus.PENDING,
            Instant.now().minus(2, MINUTES)
        );

        for (Payment p : stale) {
            try {
                PgStatus actual = pgClient.queryStatus(p.getIdempotencyKey());
                switch (actual) {
                    case PAID -> p.markPaid(actual.transactionId());
                    case FAILED -> p.markFailed(actual.errorCode());
                    case NOT_FOUND -> {
                        // 너무 일찍 polling — 다음 iteration
                        if (p.getCreatedAt().isBefore(Instant.now().minus(30, MINUTES))) {
                            p.markFailed("PG_NEVER_RECEIVED");
                        }
                    }
                    case PENDING -> {
                        // PG 도 아직 처리 중 — 더 기다림
                    }
                }
            } catch (Exception e) {
                log.error("reconcile fail for {}", p.getId(), e);
            }
            paymentRepo.save(p);
        }
    }
}
```

→ 2분 후 PENDING 인 결제는 PG status 조회. 30분 넘으면 FAILED 로 마무리 (그 사이 webhook 도 도착했을 가능성).

**(4) Webhook + scheduler 양쪽 처리** — 먼저 confirm 한 쪽이 winner

```java
@Service
public class PaymentProcessor {

    @Transactional
    public void onWebhookOrReconcile(String idempotencyKey, PgStatus pg) {
        Payment p = paymentRepo.findByIdempotencyKey(idempotencyKey)
            .orElseThrow(() -> new UnknownPayment(idempotencyKey));

        // state transition guard — 이미 확정이면 무시 (idempotent)
        if (p.isTerminal()) {
            if (p.getStatus() == PaymentStatus.PAID && pg == PgStatus.FAILED) {
                alertingService.notify("PG/DB mismatch: " + idempotencyKey);
            }
            return;
        }

        switch (pg) {
            case PAID -> p.markPaid(pg.transactionId());
            case FAILED -> p.markFailed(pg.errorCode());
            // PENDING / NOT_FOUND → 무시
        }
    }
}
```

webhook 이 먼저 도착 → DB 가 즉시 update → reconciler 가 1분 후 보면 이미 PAID → skip.
webhook 이 안 오면 → reconciler 가 PG query 로 결정.

## Frontend UX 패턴

```
1. POST /payments → 즉시 응답
   - 200 + status=PAID  → 결제 완료 화면
   - 200 + status=PENDING → "확인 중" 화면 + paymentId
   - 4xx                → 결제 실패 (재시도 안내, 같은 idempotency-key)

2. 확인 중 화면:
   - WebSocket / SSE 로 paymentId 의 상태 변화 push
   - 또는 5초 마다 GET /payments/{id} polling
   - 30초 후 안내: "처리 지연 — 카드 명세서 / 마이페이지에서 확인"
```

## 회귀 테스트

```java
@Test
void pgTimeoutShouldKeepPendingNotFailed() {
    when(pgClient.chargeWithIdempotency(any(), any()))
        .thenThrow(new PgTimeoutException("read timed out"));

    var resp = paymentService.process("key1", req);

    assertThat(resp.status()).isEqualTo(PaymentStatus.PENDING);
    var p = paymentRepo.findByIdempotencyKey("key1").orElseThrow();
    assertThat(p.getStatus()).isEqualTo(PaymentStatus.PENDING);
}

@Test
void reconcileShouldMarkPaidWhenPgConfirms() {
    var p = paymentRepo.save(Payment.pending("key2", req));
    setCreatedAt(p, Instant.now().minus(3, MINUTES));

    when(pgClient.queryStatus("key2")).thenReturn(PgStatus.paid("tx_xxx"));

    reconciler.reconcilePending();

    var refreshed = paymentRepo.findByIdempotencyKey("key2").orElseThrow();
    assertThat(refreshed.getStatus()).isEqualTo(PaymentStatus.PAID);
    assertThat(refreshed.getPgTransactionId()).isEqualTo("tx_xxx");
}

@Test
void doubleConfirmShouldBeIdempotent() {
    var p = paymentRepo.save(Payment.pending("key3", req));

    processor.onWebhookOrReconcile("key3", PgStatus.paid("tx_xxx"));   // webhook 먼저
    processor.onWebhookOrReconcile("key3", PgStatus.paid("tx_xxx"));   // reconcile 도

    var refreshed = paymentRepo.findByIdempotencyKey("key3").orElseThrow();
    assertThat(refreshed.getStatus()).isEqualTo(PaymentStatus.PAID);
    // 후처리도 1회만
}
```

## 모니터링

```
# PENDING 가 일정 시간 이상 머무름
- alert: PaymentPendingTooLong
  expr: count(payment{status="PENDING",age_minutes>10}) > 5
  for: 5m

# PG 와 DB 불일치
- alert: PaymentPgDbMismatch
  expr: payment_reconcile_mismatch_total > 0

# Reconcile 실패율
- alert: ReconcileFailureRate
  expr: rate(payment_reconcile_failed_total[5m]) > 0.1
```

## 재발 방지

- **결제 service 의 PG 호출은 timeout 명시** (connect 2s / read 5s 정도)
- **timeout 예외 ≠ FAILED** — PENDING 유지가 default. 별도 type (`PgTimeoutException`) 으로 구분
- **reconciler scheduler 가 production 에서 항상 켜져 있는지** 확인 (다른 service 의 cron 이 같은 instance 면 leader election)
- **PG status query API 인증 / rate limit** 확인 — Toss 는 30 req/min 등
- **일일 reconciliation report**: PG 의 settlement 정산 데이터 ↔ DB payment 매칭

## 관련

- [[backend]]
- [[payment-double-charge]] — idempotency-key 기반 방어
- [[payment-webhook-idempotency]] — webhook 측 idempotent 처리
- [[thread-pool-exhausted]] — 외부 호출 timeout 관련
- 책: Release It! (Stability patterns - Circuit breaker / Timeout)
