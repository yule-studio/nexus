---
title: "troubleshooting · 동시 구매 race — 재고 초과 차감 / 잔액 마이너스"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, payment, concurrency, race-condition, lock]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[payment-double-charge]]"
  - "[[deadlock-mysql]]"
---

# 동시 구매 race — 재고 초과 차감 / 잔액 마이너스

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — pessimistic / optimistic / atomic update / DB constraint |

## 증상

- 재고 5개 한정 상품에 100명 동시 구매 → 모두 결제 완료, **재고 -95**
- 한정 쿠폰 (1인 1매) 가 같은 사용자에게 여러 번 발급
- 사용자 포인트 잔액 5,000 에서 두 결제가 동시에 -3,000 차감 → 잔액 -1,000
- 선착순 이벤트 / flash sale 시 항상 발생

```sql
-- 사고 직후 발견
SELECT product_id, stock FROM product WHERE id = 1;
-- stock = -95

SELECT user_id, SUM(amount) FROM point_transaction WHERE user_id = 42;
-- -1000  (음수 → 잘못된 상태)
```

## 가설

- 결제 service 에 `if (stock > 0) stock--` 있는데 왜? → check-then-act 가 atomic 아님
- DB 트랜잭션이 격리해주지 않나? → READ COMMITTED 의 default 는 lost update 발생
- 실제: 두 트랜잭션이 같은 row 를 동시에 읽고 각자 update — 마지막 write 가 wins

## 원인 (root cause)

전형적인 lost update / check-then-act race:

```java
// Bad
@Transactional
public void purchase(Long productId, int qty) {
    Product p = productRepo.findById(productId).orElseThrow();
    if (p.getStock() < qty) throw new OutOfStock();
    p.setStock(p.getStock() - qty);   // ← 다른 트랜잭션이 동시에 같은 작업
    productRepo.save(p);
}
```

```
T1: SELECT stock FROM product WHERE id=1   → 5
T2: SELECT stock FROM product WHERE id=1   → 5
T1: 5 - 1 = 4 → UPDATE                       → 4
T2: 5 - 1 = 4 → UPDATE (overwrite!)          → 4
                                              ↑ 실제로는 2개 팔림, 재고는 4 (1개 deficit)
```

## 해결 (4가지 패턴)

### (1) Atomic UPDATE — 가장 빠름, 단순

DB 한 query 로 read + update + check 모두:

```java
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :qty " +
       "WHERE p.id = :id AND p.stock >= :qty")
int decreaseStock(@Param("id") Long id, @Param("qty") int qty);

@Transactional
public void purchase(Long productId, int qty) {
    int affected = productRepo.decreaseStock(productId, qty);
    if (affected == 0) {
        throw new OutOfStock(productId);   // 재고 부족 또는 product 없음
    }
}
```

장점: lock 없이 가장 빠름. DB 가 row-level lock 자동 적용 (UPDATE 가 짧음).
단점: stock 한 column 만 변경 — 같은 row 의 다른 변경과 함께면 다른 패턴 필요.

### (2) Pessimistic lock — 표준 방어

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}

@Transactional
public void purchase(Long productId, int qty) {
    Product p = productRepo.findByIdForUpdate(productId).orElseThrow();
    if (p.getStock() < qty) throw new OutOfStock();
    p.decreaseStock(qty);   // 같은 트랜잭션 안 보호
}
```

```sql
SELECT * FROM product WHERE id = 1 FOR UPDATE;   -- T1 lock
SELECT * FROM product WHERE id = 1 FOR UPDATE;   -- T2 → wait T1 commit
```

장점: 같은 트랜잭션 안 여러 변경 OK. 단점: throughput 감소 (직렬화). flash sale 같은 hot row 면 처리량 제한.

### (3) Optimistic lock + retry — high-concurrency 에 적합

```java
@Entity
public class Product {
    @Id Long id;
    int stock;

    @Version
    Long version;   // ← UPDATE 시 자동 WHERE version = ?

    public void decreaseStock(int qty) {
        if (stock < qty) throw new OutOfStock();
        this.stock -= qty;
    }
}

@Retryable(
    retryFor = {ObjectOptimisticLockingFailureException.class, OutOfStock.class},
    maxAttempts = 5,
    backoff = @Backoff(delay = 50, multiplier = 1.5, random = true)
)
@Transactional
public void purchase(Long productId, int qty) {
    Product p = productRepo.findById(productId).orElseThrow();
    p.decreaseStock(qty);
}
```

→ JPA 가 `UPDATE ... WHERE id=? AND version=?` 자동 생성. 충돌 시 `ObjectOptimisticLockingFailureException` → retry 가 새로 read.

장점: lock 없음 → high throughput. 단점: 충돌이 잦으면 retry 폭증 (실패도 가능 — 재고 부족이면 retry 무의미).

### (4) DB CHECK constraint — 마지막 방어

```sql
ALTER TABLE product ADD CONSTRAINT chk_stock_non_negative CHECK (stock >= 0);
ALTER TABLE point_balance ADD CONSTRAINT chk_balance_non_negative CHECK (balance >= 0);
```

→ 어떤 app 버그가 있어도 음수 차단. application 가 `CheckConstraintViolation` 받음.

```java
try {
    productRepo.save(p);
} catch (DataIntegrityViolationException e) {
    if (e.getMessage().contains("chk_stock_non_negative")) {
        throw new OutOfStock();
    }
    throw e;
}
```

## 잔액 / 포인트 도메인

```java
// Bad
@Transactional
public void usePoint(Long userId, BigDecimal amount) {
    var balance = pointRepo.findByUserId(userId);
    if (balance.compareTo(amount) < 0) throw new InsufficientPoint();
    pointRepo.decrease(userId, amount);   // ← race
}

// Good — atomic UPDATE
@Modifying
@Query("UPDATE PointBalance b SET b.balance = b.balance - :amount " +
       "WHERE b.userId = :userId AND b.balance >= :amount")
int useIfEnough(@Param("userId") Long userId, @Param("amount") BigDecimal amount);

public void usePoint(Long userId, BigDecimal amount) {
    int affected = pointRepo.useIfEnough(userId, amount);
    if (affected == 0) throw new InsufficientPoint();
}
```

추가로 **이력 기반 (event sourcing) 잔액**:

```sql
CREATE TABLE point_transaction (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    kind VARCHAR(16) NOT NULL,   -- EARN / USE
    amount DECIMAL(19,2) NOT NULL,
    balance_after DECIMAL(19,2) NOT NULL,   -- 트랜잭션 직후 잔액
    related_id BIGINT,                       -- payment_id 등
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_created (user_id, created_at)
);

-- 잔액 = SUM(amount), 또는 마지막 balance_after
-- 항상 일관성 확인 가능 (재계산)
```

## Redis 기반 분산 카운터 (선택)

flash sale 같은 hot key 에는 DB 보다 Redis atomic op 가 더 빠름:

```java
@Component
public class RedisStockService {
    private final StringRedisTemplate redis;

    public boolean tryDecrease(Long productId, int qty) {
        Long remaining = redis.opsForValue().decrement("stock:" + productId, qty);
        if (remaining < 0) {
            redis.opsForValue().increment("stock:" + productId, qty);   // 롤백
            return false;
        }
        return true;
    }
}
```

→ Redis 에 hot stock, DB 에 진실 (eventual). order INSERT 는 DB. flash sale 끝 후 reconcile.

## 회귀 테스트

```java
@Test
void concurrentPurchaseShouldRespectStock() throws Exception {
    var product = productRepo.save(Product.of("hot", BigDecimal.valueOf(10000), 10));   // stock=10
    int threads = 100;
    var executor = Executors.newFixedThreadPool(20);
    var latch = new CountDownLatch(threads);
    AtomicInteger success = new AtomicInteger();
    AtomicInteger outOfStock = new AtomicInteger();

    for (int i = 0; i < threads; i++) {
        executor.submit(() -> {
            try {
                purchaseService.purchase(product.getId(), 1);
                success.incrementAndGet();
            } catch (OutOfStock e) {
                outOfStock.incrementAndGet();
            } finally { latch.countDown(); }
        });
    }
    latch.await(10, SECONDS);

    var refreshed = productRepo.findById(product.getId()).orElseThrow();
    assertThat(refreshed.getStock()).isGreaterThanOrEqualTo(0);   // 절대 음수 X
    assertThat(success.get() + refreshed.getStock()).isEqualTo(10);   // 보존
    assertThat(success.get()).isLessThanOrEqualTo(10);
    assertThat(outOfStock.get()).isEqualTo(threads - success.get());
}

@Test
void pointBalanceShouldNeverGoNegative() throws Exception {
    var user = userRepo.save(...);
    pointRepo.save(PointBalance.of(user.getId(), BigDecimal.valueOf(5_000)));

    int threads = 50;
    var executor = Executors.newFixedThreadPool(10);
    var latch = new CountDownLatch(threads);

    for (int i = 0; i < threads; i++) {
        executor.submit(() -> {
            try { pointService.usePoint(user.getId(), BigDecimal.valueOf(200)); }
            catch (InsufficientPoint ignore) {}
            finally { latch.countDown(); }
        });
    }
    latch.await();

    var balance = pointRepo.findByUserId(user.getId());
    assertThat(balance.compareTo(BigDecimal.ZERO)).isGreaterThanOrEqualTo(0);
}
```

## 모니터링

```
- alert: NegativeStock
  expr: count(product_stock{value<0}) > 0

- alert: NegativeBalance
  expr: count(point_balance{value<0}) > 0

- alert: HighOptimisticLockRetry
  expr: rate(spring_optimistic_lock_failure_total[5m]) > 10
```

## 선택 기준

| 시나리오 | 권장 패턴 |
| --- | --- |
| 일반 재고 차감 (낮은 동시성) | (1) Atomic UPDATE |
| 한 row 에 여러 필드 변경 | (2) Pessimistic lock |
| 충돌 적음, throughput 중요 | (3) Optimistic + @Retryable |
| Flash sale / 극단 hot key | Redis (4) + DB reconcile |
| 모든 케이스 fallback | (5) DB CHECK constraint |

대부분의 결제 도메인에서 (1) + (5) 조합이 가장 simple + 안전.

## 재발 방지

- **PR 검토 룰**: stock / balance / 한정 자원 변경 시 lost update 패턴 검토 의무
- **모든 자원 column 에 CHECK constraint** (`stock >= 0`, `balance >= 0`)
- **부하 테스트**: k6 / Gatling 으로 동시 구매 시뮬레이션 → 재고 / 잔액 invariant 검증
- **이력 기반 reconcile**: 일일 batch 가 SUM(point_transaction) vs point_balance.amount 비교

## 관련

- [[backend]]
- [[payment-double-charge]] — 결제 자체 동시성
- [[deadlock-mysql]] — pessimistic lock 시 deadlock
- [[../database/database|database 트러블]]
- 책: Designing Data-Intensive Applications (Ch.7 트랜잭션 isolation level)
