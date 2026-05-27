---
title: "troubleshooting · DeadlockException (MySQL InnoDB)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, mysql, innodb, deadlock, transaction, jpa]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[../database/database|database 트러블]]"
---

# DeadlockException — `Deadlock found when trying to get lock`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — lock order / FOR UPDATE / retry 정리 |

## 증상

```
org.springframework.dao.CannotAcquireLockException:
  could not execute statement;
  SQL [n/a]; nested exception is org.hibernate.exception.LockAcquisitionException:
  could not execute statement
    Caused by: java.sql.SQLException: Deadlock found when trying to get lock;
    try restarting transaction
```

- 평소 정상, 동시 요청 spike 시 일부 트랜잭션 실패
- MySQL `SHOW ENGINE INNODB STATUS` 의 `LATEST DETECTED DEADLOCK` 섹션 확인 가능
- 재시도하면 보통 성공

## 가설

- DB 에 부하가 너무 큰가 → CPU 30%, 정상
- isolation level 가 너무 강한가 → 변경해도 deadlock 패턴 동일
- 실제: 두 트랜잭션이 같은 row 들을 **다른 순서로** lock 시도

## 원인 (root cause)

InnoDB row lock 은 트랜잭션 종료 시까지 보유. 두 트랜잭션이 lock 을 cross 한 순서로 요청하면 영원히 서로 대기 → InnoDB 가 한쪽 victim 으로 abort.

```
Tx1: UPDATE account WHERE id=1   → A lock 획득
Tx2: UPDATE account WHERE id=2   → B lock 획득
Tx1: UPDATE account WHERE id=2   → B lock 대기 (Tx2 보유)
Tx2: UPDATE account WHERE id=1   → A lock 대기 (Tx1 보유)
                                   → InnoDB: Deadlock! Tx2 abort.
```

흔한 시나리오:
- 송금 (계좌 A→B vs B→A 동시)
- 재고 차감 + 주문 생성 (순서 다르게)
- bulk update 가 임의 row 순서로 lock

## 해결

### 즉시 (workaround) — 자동 재시도

Spring Retry:
```java
@Retryable(
    retryFor = {CannotAcquireLockException.class, DeadlockLoserDataAccessException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 50, multiplier = 2, random = true)
)
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) {
    // ...
}
```

→ deadlock 은 잘 설계된 시스템에서도 가끔 발생 (정상). 재시도로 대응. 다만 자주 발생하면 근본 원인 해결 필요.

### 근본 (3가지)

**(1) Lock 순서 통일** — 가장 안전

```java
// Bad
public void transfer(Account from, Account to, BigDecimal amount) {
    from.withdraw(amount);   // from 먼저 lock
    to.deposit(amount);
}

// Good — 항상 id 작은 쪽 먼저
public void transfer(Account from, Account to, BigDecimal amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    // 명시적 lock — pessimistic
    accountRepo.findById(first.getId(), LockModeType.PESSIMISTIC_WRITE);
    accountRepo.findById(second.getId(), LockModeType.PESSIMISTIC_WRITE);

    from.withdraw(amount);
    to.deposit(amount);
}
```

```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Account findByIdForUpdate(@Param("id") Long id);
}
```

**(2) Optimistic lock + retry** — version column

```java
@Entity
public class Account {
    @Version
    private Long version;
}

// Update 시 자동으로 WHERE version = ? AND ...
// 충돌 시 OptimisticLockingFailureException → retry
```

장점: row lock 불필요 → throughput 좋음. 단점: 충돌이 잦으면 retry 폭증 (재시도가 더 비쌈).

**(3) 짧은 트랜잭션** — lock 보유 시간 ↓

```java
// Bad
@Transactional
public void process(Long id) {
    Account a = repo.findById(id);
    externalApi.call(...);   // 3초 외부 호출
    a.update(...);
}

// Good — 외부 호출 트랜잭션 밖
public void process(Long id) {
    ExternalDto d = externalApi.call(...);   // tx 밖
    updateInTx(id, d);
}

@Transactional
public void updateInTx(Long id, ExternalDto d) {
    var a = repo.findById(id);
    a.update(d);
}
```

원칙: **트랜잭션 시간 = lock 보유 시간 = deadlock 윈도우**.

## InnoDB deadlock 분석

```sql
SHOW ENGINE INNODB STATUS\G
```

→ `LATEST DETECTED DEADLOCK` 섹션:

```
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 1 sec starting index read
...
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id ... index PRIMARY of table `db`.`account`
  trx id 12345 lock_mode X locks rec but not gap

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id ... index PRIMARY of table `db`.`account`
  trx id 12345 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION: ...

*** WE ROLL BACK TRANSACTION (2)
```

→ 각 transaction 의 SQL + holds / waiting lock 확인.

permanent 로그:
```yaml
innodb_print_all_deadlocks: ON   # MySQL config
```

```sql
SHOW VARIABLES LIKE 'innodb_print_all_deadlocks';
```

## 재발 방지

- **`@Retryable` 표준 적용** — deadlock 은 정상의 일부. 5-50ms backoff + jitter 3 retry
- **PR 룰**: bulk update / multi-row transaction 추가 시 lock 순서 검토
- **테스트**: 동시 transaction 시뮬레이션
  ```java
  @Test
  void shouldHandleDeadlockGracefully() throws Exception {
      var executor = Executors.newFixedThreadPool(10);
      var futures = IntStream.range(0, 100)
          .mapToObj(i -> executor.submit(() -> service.transfer(1L, 2L, BigDecimal.ONE)))
          .toList();
      for (var f : futures) f.get();   // 모두 성공해야 함 (retry 로)
  }
  ```
- **모니터링**:
  ```
  hibernate_transactions_failed_total{reason="deadlock"}
  innodb_deadlocks (MySQL global status)
  ```

## 관련

- [[backend]]
- [[../database/database|database 트러블]] — DB 측 분석
- [[hikari-connection-leak]] — connection 영역
- [[transaction-requires-new-ignored]] — 트랜잭션 propagation
- 책: High Performance MySQL 4th Ed
