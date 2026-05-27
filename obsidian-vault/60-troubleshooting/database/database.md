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

| #  | 증상 / 에러                                                                  | 한 줄 원인                            | 핵심 해결                                                                  |
| -- | ------------------------------------------------------------------------ | --------------------------------- | ---------------------------------------------------------------------- |
| 1  | **인덱스 미사용 — full table scan** (`EXPLAIN` `type=ALL`)                      | LIKE `'%x%'` / 함수 wrap / 타입 mismatch | `LIKE 'x%'` / functional index / 컬럼 타입 일치 / covering index             |
| 2  | **Deadlock found when trying to get lock**                               | 다른 트랜잭션이 row lock 을 다른 순서로 잡음     | 잠금 순서 통일 / 짧은 트랜잭션 / `innodb_status` 분석                                |
| 3  | **Slow query — full scan with 10M+ rows**                                | 적절한 인덱스 부재 / 통계 stale             | `EXPLAIN ANALYZE` / composite index 설계 / `ANALYZE TABLE`               |
| 4  | **FATAL: too many connections** (Postgres) / **Too many connections** (MySQL) | client 가 connection pool 없이 직접 connect | client pool (HikariCP) / pgBouncer / `max_connections` 조정              |
| 5  | **Lost update — concurrent UPDATE** 같은 row                               | READ COMMITTED 에서 last-write-wins | `SELECT ... FOR UPDATE` / optimistic lock (`@Version`) / SERIALIZABLE  |
| 6  | **Phantom read — range 결과가 트랜잭션 중 바뀜**                                   | REPEATABLE READ 가 range lock 없음   | `SELECT ... FOR UPDATE` with range / SERIALIZABLE                      |
| 7  | **Postgres autovacuum 안 돌아 bloat**                                       | dead tuple 누적 / autovacuum 임계치 높음 | `VACUUM (VERBOSE, ANALYZE)` / `autovacuum_vacuum_scale_factor` 낮춤      |
| 8  | **Replication lag 폭증** (read replica)                                    | 큰 트랜잭션 / DDL / write spike        | 트랜잭션 쪼개기 / `seconds_behind_master` 모니터 / read 라우팅에 lag 임계치             |
| 9  | **`FOREIGN KEY constraint fails`**                                       | child insert 시 parent 부재 / 삭제 시 child 잔존 | order 보장 / `ON DELETE CASCADE` 명시 (의도) / soft delete 채택                |
| 10 | **Auto-increment gap** (rollback / failover 후)                            | InnoDB 가 트랜잭션 abort 후 PK 재사용 안 함  | 정상 동작 — gap 허용 / sequence (Postgres) 동일 / 순서 의미 부여 금지                  |
| 11 | **Planner 가 wrong index 선택** (Postgres / MySQL)                          | 통계가 오래되어 cardinality 오판          | `ANALYZE TABLE` / `pg_stat_statements` / `USE INDEX` hint              |
| 12 | **`ERROR 1206 (HY000): The total number of locks exceeds`** (MySQL InnoDB) | 큰 DELETE / UPDATE 가 lock 누적       | 배치 분할 (`LIMIT` loop) / `innodb_buffer_pool_size` 확대                    |
| 13 | **MySQL 5.7 → 8.0 utf8mb3 ↔ utf8mb4 충돌** (`Illegal mix of collations`)   | 컬럼별 charset 불일치                   | 전 컬럼 `utf8mb4_0900_ai_ci` 통일 / `ALTER TABLE ... CONVERT TO`            |
| 14 | **Redis OOM — `OOM command not allowed`**                                | maxmemory 도달 + eviction policy 부재 | `maxmemory-policy allkeys-lru` / TTL 부여 / hot key 분리                   |
| 15 | **Connection pool 고갈 — `Connection is not available`** (HikariCP)        | leak / blocking query 가 connection 보유 | `leak-detection-threshold` / 쿼리 timeout / pool size 조정                 |

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
