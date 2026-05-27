---
title: "troubleshooting · 결제 금액 위변조 (client-side amount trust)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, payment, security, validation]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[payment-double-charge]]"
---

# 결제 금액 위변조 — client 가 보낸 amount 신뢰

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — server-side 재계산 / signature / amount 비교 |

## 증상

- 사용자가 50,000원 상품을 100원에 결제 완료
- DevTools / Charles / mitmproxy 로 request body 의 `amount` 를 `100` 으로 조작
- 서버 / PG 모두 100원 결제 완료
- 또는: 1,000,000원 환불을 받는데 실제 결제는 50,000원 (역방향 어뷰징)

## 가설

- frontend 검증 했는데 왜? → frontend 검증은 UX, 보안 아님
- PG 가 검증해줄 것 같은데? → PG 는 client 가 보낸 amount 를 받아서 charge 할 뿐 — 상품 가격 모름
- 실제: server 가 client 의 amount 를 그대로 PG 에 전달 — 가격 신뢰의 책임 누락

## 원인 (root cause)

흔한 잘못된 흐름:

```
1. Client (frontend) → "이 상품 50,000원이에요" (UI 표시)
2. Client → POST /payments {orderId: 1, amount: 50000}
3. Server → PG.charge(amount=50000)  ← client 가 보낸 값 그대로
4. PG 가 50,000원 charge 후 응답
5. Server → payment INSERT
```

step 2 의 amount 가 공격자가 조작할 수 있는 영역. server 가 step 3 에서 자기 DB 의 상품 가격을 확인 안 하면 → 어뷰징.

```java
// Bad
@PostMapping("/payments")
public PaymentResponse pay(@RequestBody PaymentRequest req) {
    var pg = pgClient.charge(req.amount(), req.cardToken());   // ← client amount
    return new PaymentResponse(...);
}
```

## 해결

### 원칙

**Client 가 보낸 모든 값은 신뢰 불가.** 가격 / 할인 / 포인트 / 최종 금액은 **server 가 자기 DB 로 재계산**.

### 구현

**(1) Server-side amount 재계산** — 핵심

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final OrderRepository orderRepo;
    private final ProductRepository productRepo;
    private final CouponRepository couponRepo;
    private final UserPointRepository pointRepo;

    @Transactional
    public PaymentResponse process(String idempotencyKey, PaymentRequest req, Long userId) {
        // 1. 주문 조회 (소유 검증)
        Order order = orderRepo.findByIdAndUserId(req.orderId(), userId)
            .orElseThrow(() -> new OrderNotFound(req.orderId()));

        if (order.getStatus() != OrderStatus.PENDING_PAYMENT) {
            throw new InvalidOrderStatus(order.getStatus());
        }

        // 2. 가격 재계산 (server DB 의 진실)
        BigDecimal calculated = calculateActualAmount(order, req.couponId(), req.usePoint());

        // 3. client 가 보낸 금액과 일치 확인 — UX 용 검증
        if (req.amount().compareTo(calculated) != 0) {
            log.warn("amount mismatch: user={}, sent={}, calculated={}", userId, req.amount(), calculated);
            throw new AmountMismatch(req.amount(), calculated);
        }

        // 4. PG 에는 server 계산값 전달
        var pg = pgClient.chargeWithIdempotency(idempotencyKey, calculated, req.cardToken());

        // 5. payment 저장 (server 계산값 기준)
        var payment = paymentRepo.save(Payment.paid(
            idempotencyKey, order.getId(), calculated, pg.transactionId()
        ));
        order.markPaid();

        return PaymentResponse.from(payment);
    }

    private BigDecimal calculateActualAmount(Order order, Long couponId, Long usePoint) {
        BigDecimal sum = order.getItems().stream()
            .map(item -> productRepo.findById(item.productId())
                .orElseThrow()
                .getPrice()
                .multiply(BigDecimal.valueOf(item.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        if (couponId != null) {
            var coupon = couponRepo.findValidByIdAndUserId(couponId, order.getUserId())
                .orElseThrow(() -> new InvalidCoupon(couponId));
            sum = coupon.apply(sum);
        }

        if (usePoint != null && usePoint > 0) {
            var available = pointRepo.findBalance(order.getUserId());
            if (BigDecimal.valueOf(usePoint).compareTo(available) > 0) {
                throw new InsufficientPoint(usePoint, available);
            }
            sum = sum.subtract(BigDecimal.valueOf(usePoint));
        }

        if (sum.compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidAmount(sum);
        }
        return sum;
    }
}
```

→ client 의 amount 는 "내가 보고 있는 가격" 정도의 신호로만. 실제 charge 는 server 가 계산한 값.

**(2) Toss / PortOne 의 `paymentKey` + 서버 확인 패턴**

요즘 PG (Toss / PortOne) 는 2단계 확인 흐름 제공:

```
[Step 1] client → PG (직접) — 카드 정보 입력 + 1차 인증
        PG → client: paymentKey, amount

[Step 2] client → server: POST /payments/confirm {paymentKey, orderId, amount}
        server → PG.confirm(paymentKey, amount)
        PG: amount 가 step 1 과 다르면 거절
```

server 가 step 2 에서 자기 계산값 검증 + PG 가 step 1/2 amount 일치 검증 → 2중 방어.

```java
@PostMapping("/payments/confirm")
public PaymentResponse confirm(@RequestBody @Valid ConfirmRequest req, @AuthUser Long userId) {
    Order order = orderRepo.findByIdAndUserId(req.orderId(), userId).orElseThrow();
    BigDecimal expected = calculateActualAmount(order, ...);

    if (req.amount().compareTo(expected) != 0) {
        throw new AmountMismatch(req.amount(), expected);
    }

    // PG confirm — paymentKey 와 amount 가 step 1 의 것과 일치해야 통과
    var pg = tossClient.confirm(req.paymentKey(), order.getId(), expected);
    // ...
}
```

**(3) Signature / HMAC** — 변조 검증 (단순 case)

```java
// Server 가 결제 시작 시 nonce + signature 생성
public PaymentInit init(Long orderId, Long userId) {
    Order order = orderRepo.findByIdAndUserId(orderId, userId).orElseThrow();
    BigDecimal amount = calculateActualAmount(order, ...);
    String nonce = UUID.randomUUID().toString();
    String sig = HmacUtils.hmacSha256Hex(
        SERVER_SECRET,
        orderId + "|" + amount + "|" + nonce
    );
    return new PaymentInit(orderId, amount, nonce, sig);
}

// 결제 confirm 시 검증
public PaymentResponse confirm(ConfirmRequest req) {
    String expectedSig = HmacUtils.hmacSha256Hex(
        SERVER_SECRET,
        req.orderId() + "|" + req.amount() + "|" + req.nonce()
    );
    if (!MessageDigest.isEqual(expectedSig.getBytes(), req.signature().getBytes())) {
        throw new TamperedRequest();
    }
    // ...
}
```

다만 위 (1)/(2) 패턴이면 signature 필요 X — server 계산이 SSoT.

## 환불 방향 어뷰징

```java
// Bad — 환불 요청 amount 도 client 신뢰
@PostMapping("/refunds")
public RefundResponse refund(@RequestBody RefundRequest req) {
    pgClient.refund(req.paymentKey(), req.amount());   // ← 위험
}

// Good — server 가 검증
@PostMapping("/refunds")
public RefundResponse refund(@RequestBody RefundRequest req, @AuthUser Long userId) {
    Payment p = paymentRepo.findByIdAndUserId(req.paymentId(), userId).orElseThrow();
    BigDecimal already = refundRepo.sumByPaymentId(p.getId());
    BigDecimal maxRefundable = p.getAmount().subtract(already);

    if (req.amount().compareTo(maxRefundable) > 0) {
        throw new ExceedsRefundable(req.amount(), maxRefundable);
    }

    var pg = pgClient.refund(p.getPgTransactionId(), req.amount());
    refundRepo.save(Refund.from(p, req.amount(), pg.refundId()));
    // ...
}
```

## 회귀 테스트

```java
@Test
void shouldRejectTamperedAmount() {
    var order = orderRepo.save(Order.of(userId, items, BigDecimal.valueOf(50_000)));

    var req = new PaymentRequest(order.getId(), BigDecimal.valueOf(100), "tok_xxx");

    assertThatThrownBy(() -> paymentService.process("key", req, userId))
        .isInstanceOf(AmountMismatch.class);

    verify(pgClient, never()).chargeWithIdempotency(any(), any(), any());
}

@Test
void shouldRejectOrderOwnedByOtherUser() {
    var order = orderRepo.save(Order.of(otherUserId, ...));

    var req = new PaymentRequest(order.getId(), order.getAmount(), "tok_xxx");

    assertThatThrownBy(() -> paymentService.process("key", req, userId))
        .isInstanceOf(OrderNotFound.class);
}

@Test
void refundCannotExceedPayment() {
    var p = paymentRepo.save(Payment.paid("k", orderId, BigDecimal.valueOf(50_000), "tx"));

    var req = new RefundRequest(p.getId(), BigDecimal.valueOf(60_000));

    assertThatThrownBy(() -> refundService.refund(req, userId))
        .isInstanceOf(ExceedsRefundable.class);
}
```

## 모니터링

```
- alert: AmountMismatchAttempt
  expr: rate(payment_amount_mismatch_total[5m]) > 0.01
  for: 1m
  annotations:
    summary: "결제 금액 변조 시도 — 어뷰징 조사 필요"

# 너무 적은 amount
- alert: SuspiciouslySmallPayment
  expr: payment_amount_paid_total{amount<100} > 0
```

## 재발 방지

- **PR 검토 룰**: PG `charge` 호출 시 `request.getAmount()` 직접 전달 금지. 서버 계산값만.
- **ArchUnit**:
  ```java
  @Test
  void pgClientShouldOnlyReceiveServerCalculatedAmount() {
      // PgClient 의 charge 메서드 호출 시 amount 인자는 직접 request 에서 오면 안 됨
  }
  ```
- **모든 결제 endpoint 에 `@AuthUser` 의무화** — 인증 없는 결제 endpoint 금지
- **금액 column 은 `DECIMAL` 사용** — `double` / `float` 는 정확도 손실 위험
  ```java
  @Column(precision = 19, scale = 2)
  private BigDecimal amount;   // ← 절대 double 아님
  ```
- **OWASP ASVS L2 검증** — 결제 도메인은 보안 standard 준수

## 관련

- [[backend]]
- [[payment-double-charge]] — 동시성 방어
- [[spring-security-401]] — 인증 / 권한
- 책: OWASP Top 10 / OWASP ASVS
