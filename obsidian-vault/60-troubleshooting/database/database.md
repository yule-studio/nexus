---
title: "60-troubleshooting/database — Database 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, database, mysql, postgres, index, deadlock, hub]
home_hub: "60-troubleshooting"
---

# Database 트러블 카탈로그

> MySQL / Postgres / Redis 운영에서 자주 만나는 대표 트러블.

**[[../troubleshooting|↑ 60-troubleshooting]]** · **[[../../20-areas/database/database|↑↑ 20-areas/database]]**

---

## 카탈로그

| #  | 증상 / 에러                                                                  | 한 줄 원인                            | 핵심 해결                                                                  | 깊이 노트                                            |
| -- | ------------------------------------------------------------------------ | --------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------ |
| 1  | **인덱스 미사용 — full table scan**                                            | LIKE `'%x%'` / 함수 wrap / 타입 mismatch | `LIKE 'x%'` / functional index / 컬럼 타입 일치 / covering index             | [[index-not-used]]                               |
| 2  | **Deadlock found when trying to get lock**                               | 다른 트랜잭션이 row lock 을 다른 순서로 잡음     | 잠금 순서 통일 / @Retryable / 짧은 트랜잭션                                       | [[../backend/deadlock-mysql]]                    |
| 3  | **Slow query — full scan with 10M+ rows**                                | 적절한 인덱스 부재 / 통계 stale             | EXPLAIN ANALYZE / composite + covering index / keyset pagination       | [[slow-query]]                                   |
| 4  | **FATAL: too many connections**                                          | client 가 connection pool 없이 직접 connect | pgBouncer / ProxySQL / Hikari pool 적절히 / max_connections 산정          | [[too-many-connections]]                         |
| 5  | **Lost update — concurrent UPDATE 같은 row**                               | READ COMMITTED 에서 last-write-wins | Atomic UPDATE / pessimistic / optimistic (@Version)                   | [[lost-update]]                                  |
| 6  | **Phantom read — range 결과가 트랜잭션 중 바뀜**                                   | REPEATABLE READ 가 range lock 없음   | gap lock / SERIALIZABLE / Unique constraint                            | [[phantom-read]]                                 |
| 7  | **Postgres autovacuum 안 돌아 bloat**                                       | dead tuple 누적 / long-running tx   | autovacuum scale_factor 낮춤 / VACUUM (VERBOSE) / long tx 차단           | [[postgres-autovacuum-bloat]]                    |
| 8  | **Replication lag 폭증**                                                   | 큰 트랜잭션 / DDL / write spike        | multi-threaded replication / 트랜잭션 split / lag 기반 라우팅                 | [[replication-lag]]                              |
| 9  | **`FOREIGN KEY constraint fails`**                                       | child insert 시 parent 부재 / 삭제 시 child 잔존 | order 보장 / ON DELETE CASCADE / soft delete / deferred constraint  | [[foreign-key-violation]]                        |
| 10 | **Auto-increment gap**                                                   | InnoDB 가 트랜잭션 abort 후 PK 재사용 X    | 정상 동작 — id 는 unique 일 뿐, 노출 order_no 별도 컬럼                            | [[auto-increment-gap]]                           |
| 11 | **Planner 가 wrong index 선택**                                             | 통계가 오래되어 cardinality 오판          | ANALYZE TABLE / extended statistics / index hint (최후)                  | [[planner-wrong-index]]                          |
| 12 | **`ERROR 1206` — total locks exceeds** (InnoDB)                          | 큰 DELETE / UPDATE 가 lock 누적       | LIMIT loop / partitioning DROP PARTITION / pt-online-schema-change    | [[innodb-locks-exceeded]]                        |
| 13 | **`Illegal mix of collations`** (utf8mb3 ↔ utf8mb4)                      | 컬럼별 charset 불일치                   | utf8mb4_0900_ai_ci 통일 / ALTER TABLE CONVERT TO / JDBC URL 명시          | [[mysql-utf8mb4-collation]]                      |
| 14 | **Redis OOM — `OOM command not allowed`**                                | maxmemory 도달 + eviction policy 부재 | maxmemory-policy allkeys-lru / TTL 의무 / bigkeys 분석                   | [[redis-oom]]                                    |
| 15 | **HikariPool — Connection is not available**                             | leak / 트랜잭션 안 외부 호출              | try-with-resources / leak-detection-threshold / 트랜잭션 안 IO 금지         | [[../backend/hikari-connection-leak]]            |

## 자주 쓰는 진단 명령

```sql
-- MySQL
SHOW ENGINE INNODB STATUS\G   -- deadlock detail
SHOW PROCESSLIST;
EXPLAIN ANALYZE SELECT ...;

-- Postgres
SELECT * FROM pg_stat_activity WHERE state != 'idle';
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
SELECT * FROM pg_stat_user_tables WHERE n_dead_tup > 10000;

-- Redis
INFO memory
CLIENT LIST
SLOWLOG GET 10
```

## 관련

- [[../troubleshooting]]
- [[../../20-areas/database/database|20-areas/database]]
- [[../backend/backend|backend 트러블]] — JDBC / Hikari / JPA 경계
