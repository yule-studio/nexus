---
title: "troubleshooting · Lost update — concurrent UPDATE 같은 row"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, transaction, concurrency, isolation]
home_hub: "60-troubleshooting"
related: ["[[database]]", "[[../backend/payment-race-stock-balance]]"]
---

# Lost update — 두 트랜잭션이 같은 row 동시 UPDATE

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- 같은 row 를 두 트랜잭션이 동시에 update → 한쪽 변경이 사라짐
- 재고 / 잔액 / counter 가 예상과 다름
- READ COMMITTED (default) 에서 발생

```
T1: SELECT balance → 1000
T2: SELECT balance → 1000
T1: UPDATE balance = 1000 - 300 = 700
T2: UPDATE balance = 1000 - 200 = 800   ← T1 변경 lost
```

## 해결

### Atomic UPDATE
```sql
UPDATE account SET balance = balance - :amount WHERE id = :id AND balance >= :amount;
-- 1 query 안에 read + update + check
```

### Pessimistic lock
```sql
SELECT * FROM account WHERE id = :id FOR UPDATE;
-- 다른 트랜잭션은 wait
```

### Optimistic lock (`@Version`)
```java
@Entity
public class Account {
    @Version Long version;   // UPDATE 시 자동 WHERE version=?
}
```
→ 충돌 시 `ObjectOptimisticLockingFailureException` + retry

### REPEATABLE READ / SERIALIZABLE
isolation level 강화. throughput 영향 → 보통 위 3가지 우선.

## 진단

```sql
-- 같은 row 의 update history 추적 (audit table 필요)
SELECT row_id, value, updated_at, updated_by FROM account_audit
WHERE row_id = ? ORDER BY updated_at;
```

## 관련

- [[database]] / [[../backend/payment-race-stock-balance]] / [[../backend/deadlock-mysql]]
