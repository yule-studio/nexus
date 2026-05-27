---
title: "troubleshooting · Planner 가 wrong index 선택"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, planner, statistics]
home_hub: "60-troubleshooting"
related: ["[[database]]", "[[index-not-used]]", "[[slow-query]]"]
---

# Planner 가 wrong index 선택 — 통계 stale

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- 어제까지 빠르던 쿼리가 갑자기 느림
- 새 데이터 대량 INSERT 후
- `EXPLAIN` 에 의도 안 한 인덱스 사용
- analyze 한 번 돌리면 정상

## 원인 (root cause)

DB planner 는 통계 (cardinality / histogram / row count) 기반 cost estimate. 데이터 분포 급변 + 통계 stale → 잘못된 plan 선택:

```sql
-- 통계 기준 5,000 row 예상 → index scan 결정
-- 실제 5M row → 5M 회 random IO → 매우 느림
```

## 해결

### 통계 갱신
```sql
-- MySQL
ANALYZE TABLE orders;

-- Postgres
ANALYZE orders;
-- 컬럼별
ANALYZE orders (user_id, created_at);
```

### Sampling rate 조정 (Postgres)
```sql
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 1000;   -- default 100
ANALYZE orders;
```

→ 더 많은 sample 으로 정확한 histogram.

### Index hint (마지막 수단)
```sql
-- MySQL
SELECT * FROM orders USE INDEX (idx_user_created) WHERE user_id = ? AND created_at > ?;
SELECT * FROM orders FORCE INDEX (idx_user_created) WHERE ...;

-- Postgres — pg_hint_plan extension
/*+ IndexScan(orders idx_user_created) */
SELECT * FROM orders WHERE ...;
```

→ 가능하면 통계 / 인덱스 설계로 해결. hint 는 brittle.

### Multi-column statistics (Postgres extended statistics)
```sql
CREATE STATISTICS s_orders_user_status (dependencies, ndistinct)
ON user_id, status FROM orders;
ANALYZE orders;
```

→ user_id 와 status 가 correlated 일 때 정확한 row count 추정.

### Plan cache 무력화
```sql
-- prepared statement plan cache 가 stale plan 보유
DEALLOCATE ALL;   -- Postgres
RESET QUERY CACHE;   -- MySQL (deprecated MySQL 8+)
```

JPA / Hibernate 의 query plan cache:
```yaml
spring.jpa.properties.hibernate.query.plan_cache_max_size: 2048
spring.jpa.properties.hibernate.query.plan_parameter_metadata_max_size: 128
```

## 진단

```sql
-- MySQL
EXPLAIN FORMAT=JSON SELECT ...;   -- cost / row 추정 vs 실제
EXPLAIN ANALYZE SELECT ...;        -- 실제 시간

-- Postgres
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
-- "rows=X (estimated)" vs "rows=Y (actual)" 비교
-- 차이 10x 이상 → 통계 stale 의심

-- Stat 마지막 시점
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables;
```

## 재발 방지

- autovacuum + auto-analyze 켜기 (Postgres default)
- 대량 INSERT / DELETE 후 manual `ANALYZE`
- 모니터링: row estimate vs actual 차이 알람
- 정기 plan review (slow query log + EXPLAIN)

## 관련

- [[database]] / [[index-not-used]] / [[slow-query]] / [[postgres-autovacuum-bloat]]
