---
title: "troubleshooting · PG webhook 재전송으로 중복 처리"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, payment, webhook, idempotency]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[payment-double-charge]]"
---

# PG webhook 재전송으로 중복 처리 — 포인트 2배 적립 / 배송 2번 등록

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — webhook event_id / state machine / dedup |

## 증상

- PG (Toss / PortOne / Stripe / KakaoPay) 가 같은 결제 완료 webhook 을 여러 번 발진
- 결과: 사용자 포인트 2배 적립 / 알림 2회 발송 / 배송 2건 생성 / 통계 부풀려짐
- 로그에 같은 `transactionId` webhook event 2-3개

```
[INFO] webhook received: orderId=123, txId=tx_abc, event=PAID  ← 1st
[INFO] webhook received: orderId=123, txId=tx_abc, event=PAID  ← 2nd (3초 후)
[INFO] webhook received: orderId=123, txId=tx_abc, event=PAID  ← 3rd
```

## 가설

- PG 가 실수로 보냄? → 정상 동작 — webhook 은 **at-least-once** delivery
- HTTP 200 으로 응답했는데 왜 또? → response timeout / TCP 끊김 / PG 의 ack 미수신
- 실제: webhook 은 본질적으로 재전송될 수 있는 것 — 수신측이 idempotent 여야 함

## 원인 (root cause)

webhook 은 분산 시스템의 message delivery. 다음 시나리오 모두 재전송 trigger:

1. **HTTP timeout** — server 가 5초 후 응답, PG 는 3초에 끊고 재시도
2. **5xx 응답** — server bug 또는 LB 장애 시 PG 가 backoff 후 재시도
3. **Network partition** — 응답이 PG 까지 도달 못함
4. **PG 내부 재처리** — PG 의 internal retry queue 가 다시 emit
5. **수동 재발진** — PG 운영자가 missed 의심 시 재전송

비즈니스 처리가 idempotent 가 아니면 (`points += 1000` / `INSERT INTO shipment ...`) 매 호출마다 부작용 누적.

```java
// Bad — non-idempotent
@PostMapping("/webhooks/payment")
public void onPaymentWebhook(@RequestBody PgWebhook event) {
    var payment = paymentRepo.findByPgTransactionId(event.transactionId()).orElseThrow();
    payment.markPaid();
    pointService.addPoints(payment.userId(), 1000);   // ← 매번 +1000
    shipmentService.create(payment.orderId());        // ← 매번 새 shipment
    notificationService.send(payment.userId(), "결제 완료");   // ← 매번 알림
}
```

## 해결

### 즉시 (workaround)

burst webhook 만 dedupe 하는 in-memory cache (5분):

```java
private final Cache<String, Boolean> recent = Caffeine.newBuilder()
    .expireAfterWrite(5, MINUTES).maximumSize(10000).build();

if (recent.getIfPresent(event.id()) != null) {
    return;  // skip
}
recent.put(event.id(), true);
```

→ 단일 instance / 단시간 한정. 여러 pod / 며칠 후 재전송 막을 수 없음.

### 근본 (3가지 layer)

**(1) `webhook_event` 테이블에 처리 기록 + unique constraint**

```sql
CREATE TABLE webhook_event (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    provider VARCHAR(32) NOT NULL,        -- toss / portone / stripe
    event_id VARCHAR(128) NOT NULL,        -- PG 가 보낸 unique id
    event_type VARCHAR(64) NOT NULL,       -- PAID / CANCELED / REFUNDED
    payload JSON NOT NULL,
    received_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL,
    UNIQUE KEY uk_provider_event (provider, event_id)
);
```

```java
@RestController
public class PaymentWebhookController {
    private final WebhookEventRepository eventRepo;
    private final PaymentProcessor processor;

    @PostMapping("/webhooks/payment/toss")
    public ResponseEntity<Void> onToss(@RequestBody String rawBody,
                                       @RequestHeader("Toss-Signature") String sig) {
        if (!verifySignature(rawBody, sig)) return ResponseEntity.status(401).build();

        var event = parse(rawBody);
        WebhookEvent record;
        try {
            record = eventRepo.save(WebhookEvent.received("toss", event.eventId(), event.type(), rawBody));
        } catch (DataIntegrityViolationException dup) {
            // 이미 처리됐거나 처리 중 → 200 응답 (PG 가 재시도 멈춤)
            return ResponseEntity.ok().build();
        }

        // 비즈니스 처리 (idempotent 추가 layer 도 있음)
        processor.process(event);
        record.markProcessed();
        eventRepo.save(record);

        return ResponseEntity.ok().build();
    }
}
```

**(2) 비즈니스 처리도 idempotent — state machine + version check**

```java
@Service
public class PaymentProcessor {
    @Transactional
    public void process(PgWebhook event) {
        Payment payment = paymentRepo.findByPgTransactionId(event.transactionId()).orElseThrow();

        // state transition guard — PENDING → PAID 만 허용
        if (payment.getStatus() == PaymentStatus.PAID) {
            log.info("already paid: {}", event.transactionId());
            return;  // idempotent
        }

        if (payment.getStatus() != PaymentStatus.PENDING) {
            throw new IllegalStateException("cannot transition from " + payment.getStatus());
        }

        payment.markPaid();
        eventPublisher.publishEvent(new PaymentPaidEvent(payment.getId()));   // 후처리 분리
    }
}

// 후처리도 각자 idempotent
@Component
public class PointAfterPaymentListener {
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onPaid(PaymentPaidEvent e) {
        // point_transaction 에 unique (payment_id, kind='EARN') → 중복 INSERT 차단
        try {
            pointTxRepo.save(PointTransaction.earn(e.paymentId(), 1000));
        } catch (DataIntegrityViolationException dup) {
            log.info("point already earned: {}", e.paymentId());
        }
    }
}
```

**(3) Outbox + outbox dedup** — 큰 시스템에서

webhook → outbox (PG event 영속) → consumer (idempotent) 패턴. 비즈니스 처리 실패해도 outbox 에 보존 → 재처리 가능.

```
1. webhook 수신 → webhook_event INSERT (HTTP 200 즉시 반환)
2. background poller → unprocessed event → 비즈니스 처리 → mark processed
3. 처리 실패 → retry (max attempts 후 dead letter)
```

장점: HTTP timeout 차단 (즉시 200), 비즈니스 처리 시간이 webhook 응답에 영향 X.

## 서명 검증 함정

webhook endpoint 는 public — 누구나 fake 요청 가능. **서명 검증 필수**:

```java
public boolean verifySignature(String rawBody, String signatureHeader) {
    String expected = HmacUtils.hmacSha256Hex(WEBHOOK_SECRET, rawBody);
    return MessageDigest.isEqual(
        expected.getBytes(),
        signatureHeader.getBytes()
    );
}
```

주의:
- `rawBody` 는 Jackson 파싱 전 원본 (whitespace 보존). `@RequestBody String` 또는 filter 에서 capture
- `MessageDigest.isEqual` — timing attack 방어 (정확한 length 비교)
- secret 은 환경 변수 / Secrets Manager

## webhook 처리 SLA

PG 정책 예시:
- Toss: HTTP 200 응답 안 오면 30초 후 1회 retry, 그 후 5분 / 30분 / ... 최대 24시간
- Stripe: exponential backoff, 최대 3일 (`evt_*` event 영구 보존)
- KakaoPay: 5분 마다 5회 retry

→ **200 빠르게 응답** (비즈니스 처리는 async 권장). 처리에 30초 걸리면 PG 가 retry 시작 → dedup 부하 ↑.

## 회귀 테스트

```java
@Test
void duplicateWebhookShouldNotDoubleProcess() {
    var event = new PgWebhook("evt_123", "tx_abc", "PAID", ...);

    processor.process(event);   // 1st
    processor.process(event);   // 2nd

    // 포인트는 1회만 적립
    var points = pointTxRepo.findByPaymentId(payment.getId());
    assertThat(points).hasSize(1);
    assertThat(points.get(0).getAmount()).isEqualTo(1000);

    // 알림도 1회만
    verify(notificationClient, times(1)).send(any());
}

@Test
void webhookEventIdShouldBeUniqueAcrossInstances() throws Exception {
    var event = new PgWebhook("evt_dup", "tx_abc", "PAID", ...);
    int n = 10;
    var executor = Executors.newFixedThreadPool(10);
    var latch = new CountDownLatch(n);

    for (int i = 0; i < n; i++) {
        executor.submit(() -> {
            try {
                controller.onToss(event.rawBody(), event.signature());
            } catch (Exception ignore) {
            } finally { latch.countDown(); }
        });
    }
    latch.await();

    assertThat(eventRepo.count()).isEqualTo(1);
}
```

## 진단

```sql
-- 같은 event 가 여러 번 수신됨
SELECT event_id, COUNT(*)
FROM webhook_event
WHERE received_at > NOW() - INTERVAL 1 DAY
GROUP BY event_id
HAVING COUNT(*) > 1;
-- → unique constraint 동작했으면 항상 1

-- 처리 안 된 webhook
SELECT * FROM webhook_event
WHERE processed_at IS NULL
  AND received_at < NOW() - INTERVAL 10 MINUTE;
-- → 처리 실패 / async backlog 표면화

-- 같은 payment 의 후속 이벤트 (point/shipment) 중복
SELECT payment_id, COUNT(*)
FROM point_transaction
WHERE kind = 'EARN'
GROUP BY payment_id
HAVING COUNT(*) > 1;
```

## 재발 방지

- **`webhook_event(provider, event_id)` unique constraint** — 모든 PG webhook 의 첫 방어
- **state machine** — `Payment.status` 가 unidirectional (PENDING → PAID → REFUNDED, 역 X)
- **모든 후처리에 unique key** — `point_transaction(payment_id, kind)` / `shipment(order_id)` / `notification(payment_id, template_id)`
- **서명 검증 필수** — `@PostMapping` 진입 즉시
- **알람**:
  ```
  - alert: WebhookProcessingBacklog
    expr: count(webhook_event{processed_at=null,received_at<NOW()-10m}) > 0
  ```
- **PG 의 webhook log** 와 DB `webhook_event` count 일일 reconciliation

## 관련

- [[backend]]
- [[payment-double-charge]] — 결제 자체 중복
- [[payment-pg-timeout-uncertainty]] — webhook 도착 전 client timeout
- [[../../20-areas/backend/spring/spring|20-areas/spring]]
