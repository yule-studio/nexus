---
title: "troubleshooting · Double charge — 동시 요청 시 중복 결제"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, payment, concurrency, idempotency, race-condition]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[payment-webhook-idempotency]]"
  - "[[payment-pg-timeout-uncertainty]]"
---

# Double charge — 동시 요청 시 같은 주문이 2회 결제

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — idempotency-key / unique constraint / pessimistic lock |

## 증상

- 사용자 1명이 같은 주문에 대해 PG (Toss / PortOne / NicePay) 영수증 2장 수령
- 카드 명세서에 같은 금액 2회 charge
- DB `payment` 테이블에 같은 `order_id` 로 row 2개 (status=`PAID`)
- 보통 발생 패턴:
  1. **사용자 더블 클릭** — 결제 버튼 빠르게 2번
  2. **네트워크 retry** — client / proxy / mobile 가 응답 timeout 후 자동 재전송
  3. **PG webhook + client confirm 양쪽 도착** — race
  4. **Load balancer 가 같은 request 를 2개 서버로** (드물지만 healthcheck 전환 시점)

## 가설

- frontend 가 버튼 disable 하면 해결? → 다른 디바이스 / 새로고침 / 안드 background retry 막을 수 없음
- DB unique constraint 만 걸면 해결? → 두 트랜잭션이 동시에 INSERT 시도 → 한쪽은 `DuplicateKeyException` 인데 **이미 PG 에 charge 는 됐음** → 환불 필요
- 실제: PG 호출 자체를 idempotent 하게 만들어야 함

## 원인 (root cause)

결제 처리는 보통:
```
1. POST /payments  (client 가 결제 요청)
2. server → PG.charge(card, amount)   ← 외부 호출 + side effect
3. server → DB.payment INSERT
4. response → client
```

step 2 가 **외부 부수 효과** (실제 돈이 빠짐). 동시 2 request 가 step 2 를 동시에 통과 → 같은 카드로 2회 charge → step 3 에서 DB unique constraint 가 한쪽을 거부해도 PG 는 이미 2건.

```java
// Bad — race condition
@Transactional
public Payment process(PaymentRequest req) {
    var order = orderRepo.findById(req.orderId()).orElseThrow();

    // 다른 thread 가 같은 order 로 동시에 진입 가능
    if (paymentRepo.findByOrderId(req.orderId()).isPresent()) {
        throw new AlreadyPaid();
    }

    PgResult pg = pgClient.charge(order.amount(), req.cardToken());   // ← side effect

    return paymentRepo.save(new Payment(order, pg.transactionId()));
}
```

`findByOrderId` 와 `charge` 사이에 다른 트랜잭션 가능 → 양쪽 다 통과 → 양쪽 다 charge.

## 해결

### 즉시 (workaround)

운영 hotfix 가 어려움. 일단 환불 + 사용자 알림 + reconciliation script. 근본 수정 배포 후에야 안전.

### 근본 (4 layer 방어)

**(1) Client 에 `Idempotency-Key` 헤더 요구** — 핵심

```http
POST /payments
Idempotency-Key: 9f2e8c1a-3b4d-4f5e-8a7b-1c2d3e4f5a6b
Content-Type: application/json

{"orderId": 123, "cardToken": "tok_xxx", "amount": 50000}
```

client (frontend / mobile) 가 결제 화면 진입 시 1회 UUID 생성 → 같은 화면의 재시도는 같은 key. 다른 화면 / 다른 주문은 다른 key.

server 측 처리:

```java
@RestController
@RequiredArgsConstructor
public class PaymentController {
    private final PaymentService paymentService;

    @PostMapping("/payments")
    public PaymentResponse pay(
        @RequestHeader("Idempotency-Key") @NotBlank String idempotencyKey,
        @RequestBody @Valid PaymentRequest req
    ) {
        return paymentService.processIdempotent(idempotencyKey, req);
    }
}
```

**(2) DB unique constraint on `idempotency_key`** — 빠른 거절

```sql
CREATE TABLE payment (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    idempotency_key VARCHAR(64) NOT NULL,
    pg_transaction_id VARCHAR(64),
    amount DECIMAL(19,2) NOT NULL,
    status VARCHAR(16) NOT NULL,    -- PENDING / PAID / FAILED
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_idempotency (idempotency_key),
    UNIQUE KEY uk_order (order_id)
);
```

**(3) `PENDING` row 선등록 패턴** — PG 호출 전에 DB 에 점유 표시

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentRepository paymentRepo;
    private final OrderRepository orderRepo;
    private final PgClient pgClient;

    public PaymentResponse processIdempotent(String key, PaymentRequest req) {
        // 1. PENDING row INSERT — 같은 key 동시 진입은 unique constraint 로 차단
        Payment payment;
        try {
            payment = paymentRepo.save(Payment.pending(key, req.orderId(), req.amount()));
        } catch (DataIntegrityViolationException dup) {
            // 동시 진입한 다른 request → 기존 row 조회 후 동일 응답 반환 (idempotent)
            return loadResult(key);
        }

        // 2. PG 호출 (이 시점에 같은 key 는 다른 thread 가 절대 동시 호출 못함)
        try {
            PgResult pg = pgClient.chargeWithIdempotency(key, req.cardToken(), req.amount());
            payment.markPaid(pg.transactionId());
        } catch (PgException e) {
            payment.markFailed(e.getCode());
            // 주의: timeout 이면 PENDING 유지 + reconciliation 으로
            // [[payment-pg-timeout-uncertainty]] 참조
        }
        paymentRepo.save(payment);

        return PaymentResponse.from(payment);
    }

    private PaymentResponse loadResult(String key) {
        // PENDING 이면 wait or 202 응답 (client 가 polling)
        var existing = paymentRepo.findByIdempotencyKey(key).orElseThrow();
        return PaymentResponse.from(existing);
    }
}
```

**(4) PG 측 idempotency-key 도 같이 전달** — PG 도 중복 charge 방지

Toss / PortOne / Stripe 등 주요 PG 가 `Idempotency-Key` 헤더 지원. 같은 key 로 2회 호출 → PG 가 첫 결과만 반환 (2번째 charge 안 함).

```java
@Component
public class TossPaymentClient implements PgClient {
    private final RestClient restClient;

    @Override
    public PgResult chargeWithIdempotency(String idempotencyKey, String cardToken, BigDecimal amount) {
        return restClient.post()
            .uri("https://api.tosspayments.com/v1/payments/confirm")
            .header("Idempotency-Key", idempotencyKey)    // ← PG 도 인식
            .header("Authorization", "Basic " + secret)
            .body(Map.of("paymentKey", cardToken, "amount", amount))
            .retrieve()
            .body(PgResult.class);
    }
}
```

→ DB unique + server PENDING + PG idempotency-key = 3중 방어. 어느 1개만 실패해도 다른 layer 가 막음.

## 다른 방어 패턴 (보조)

### Pessimistic lock on order

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT o FROM Order o WHERE o.id = :id")
    Optional<Order> findByIdForUpdate(@Param("id") Long id);
}

@Transactional
public PaymentResponse process(...) {
    Order o = orderRepo.findByIdForUpdate(req.orderId()).orElseThrow();
    if (o.isPaid()) return existing(o);
    // ...
}
```

같은 order 에 대해서는 직렬화. 다만 high-concurrency 면 throughput 영향 → idempotency-key + unique constraint 가 우선.

### Redis distributed lock (Redisson)

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final RedissonClient redisson;

    public PaymentResponse process(String key, PaymentRequest req) {
        RLock lock = redisson.getLock("payment:lock:" + key);
        try {
            if (!lock.tryLock(2, 10, TimeUnit.SECONDS)) {
                throw new TooManyConcurrentRequests();
            }
            // critical section
        } finally {
            if (lock.isHeldByCurrentThread()) lock.unlock();
        }
    }
}
```

cross-instance idempotency 필요할 때. 다만 분산락은 fencing token 없으면 GC pause / network partition 시 부정확 — DB unique constraint 가 fallback.

### Outbox pattern + transactional message

```
1. Payment INSERT (DB) + outbox INSERT — 같은 트랜잭션
2. background poller → outbox → PG.charge
3. PG 응답 → Payment 상태 update
```

PG 호출이 트랜잭션 밖. retry / replay 안전. 다만 latency (사용자가 즉시 결과 못 받음) → 결제 같은 동기 도메인에는 잘 맞지 않을 수도.

## 회귀 테스트

```java
@SpringBootTest
@AutoConfigureMockMvc
class PaymentConcurrencyTest {

    @Autowired MockMvc mockMvc;
    @MockBean PgClient pgClient;
    @Autowired PaymentRepository paymentRepo;

    @Test
    void sameIdempotencyKeyConcurrent_shouldChargeOnce() throws Exception {
        when(pgClient.chargeWithIdempotency(any(), any(), any()))
            .thenReturn(new PgResult("tx_123"));

        String key = UUID.randomUUID().toString();
        int n = 100;
        var executor = Executors.newFixedThreadPool(20);
        var latch = new CountDownLatch(n);

        for (int i = 0; i < n; i++) {
            executor.submit(() -> {
                try {
                    mockMvc.perform(post("/payments")
                        .header("Idempotency-Key", key)
                        .contentType(APPLICATION_JSON)
                        .content("""{"orderId":1,"cardToken":"x","amount":50000}"""))
                        .andExpect(status().is2xxSuccessful());
                } finally { latch.countDown(); }
            });
        }
        latch.await(10, TimeUnit.SECONDS);

        // PG 는 1회만 호출, DB 도 1 row
        verify(pgClient, times(1)).chargeWithIdempotency(eq(key), any(), any());
        assertThat(paymentRepo.findByIdempotencyKey(key)).isPresent();
        assertThat(paymentRepo.count()).isEqualTo(1);
    }
}
```

## 진단 / 사고 분석

```sql
-- 같은 order 2회 결제 발견
SELECT order_id, COUNT(*), GROUP_CONCAT(pg_transaction_id)
FROM payment
WHERE status = 'PAID'
GROUP BY order_id
HAVING COUNT(*) > 1;

-- 같은 사용자가 짧은 시간에 결제 시도
SELECT user_id, COUNT(*), MIN(created_at), MAX(created_at)
FROM payment
WHERE created_at > NOW() - INTERVAL 1 HOUR
GROUP BY user_id
HAVING COUNT(*) > 3;

-- 시간차 분석 — 동시 진입 패턴
SELECT * FROM payment
WHERE order_id = ?
ORDER BY created_at;
-- 두 row 의 created_at 차이가 ms 단위 → race condition 증거
```

## 재발 방지

- **결제 API 의 `Idempotency-Key` 헤더는 필수** — 누락 시 400 Bad Request
- **DB unique constraint** 2개: `(idempotency_key)` + `(order_id, status='PAID')` (partial / functional index)
- **PG 측 idempotency-key 활용** 확인 — Toss / PortOne / Stripe 모두 지원
- **부하 테스트**: k6 / Gatling 으로 같은 idempotency-key 100 동시 → 결제 1회만 발생 검증
- **모니터링 알람**:
  ```
  - alert: DoublePaymentDetected
    expr: count by(order_id)(payment_paid_total) > 1
    for: 1m
  ```
- **일일 reconciliation**: PG 의 transaction list ↔ DB payment list 매칭 (mismatch → 즉시 알람)

## 관련

- [[backend]]
- [[payment-webhook-idempotency]] — webhook 재처리 시 중복
- [[payment-pg-timeout-uncertainty]] — PG 응답 불명 시
- [[payment-race-stock-balance]] — 재고 / 잔액 race
- [[deadlock-mysql]] — pessimistic lock 사용 시
- 책: Designing Data-Intensive Applications (Ch.7 트랜잭션 / Ch.9 일관성)
