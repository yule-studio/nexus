---
title: "troubleshooting · Phantom read — range 결과가 트랜잭션 중 바뀜"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, transaction, isolation, phantom]
home_hub: "60-troubleshooting"
related: ["[[database]]", "[[lost-update]]"]
---

# Phantom read — 같은 query 가 다른 row count 반환

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

같은 트랜잭션 안에서 같은 `SELECT ... WHERE` 쿼리가 다른 결과:

```
T1: SELECT COUNT(*) FROM reservation WHERE seat = 'A1';   → 0
T2: INSERT INTO reservation (seat) VALUES ('A1'); COMMIT;
T1: SELECT COUNT(*) FROM reservation WHERE seat = 'A1';   → 1  (phantom!)
T1: INSERT INTO reservation (seat) VALUES ('A1');   ← 같은 좌석 2건 예약
```

## 원인

isolation level 별 phantom 허용:

| Level | Dirty | Non-repeatable | Phantom |
| --- | --- | --- | --- |
| READ UNCOMMITTED | ❌ | ❌ | ❌ |
| READ COMMITTED (default) | ✅ | ❌ | ❌ |
| REPEATABLE READ (MySQL default) | ✅ | ✅ | MySQL ✅ / Postgres ❌* |
| SERIALIZABLE | ✅ | ✅ | ✅ |

*Postgres REPEATABLE READ 는 snapshot isolation → 대부분의 phantom 방어.

## 해결

### Range lock (`SELECT ... FOR UPDATE`)
```sql
BEGIN;
SELECT * FROM reservation WHERE seat = 'A1' FOR UPDATE;   -- gap + row lock
-- ... 비즈니스 검증
INSERT INTO reservation (seat) VALUES ('A1');
COMMIT;
```

MySQL InnoDB 는 gap lock 으로 range 까지 보호. Postgres 는 row 만 → SERIALIZABLE 필요.

### SERIALIZABLE isolation
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void reserve(...) { ... }
```

Postgres 는 SSI (Serializable Snapshot Isolation) — 충돌 감지 시 `SerializationFailure` → retry.

### Unique constraint — 가장 단순
```sql
ALTER TABLE reservation ADD CONSTRAINT uk_seat UNIQUE (seat, event_id);
-- 2번째 INSERT 가 DuplicateKeyException → 같은 좌석 2건 차단
```

```java
@Retryable(retryFor = DataIntegrityViolationException.class)
public void reserve(...) { ... }
```

## 진단

```sql
-- MySQL InnoDB lock 확인
SHOW ENGINE INNODB STATUS\G   -- LOCK section

-- Postgres
SELECT * FROM pg_locks WHERE NOT granted;
```

## 관련

- [[database]] / [[lost-update]] / [[../backend/deadlock-mysql]]
