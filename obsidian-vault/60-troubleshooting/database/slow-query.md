---
title: "troubleshooting · Slow query (full scan / planner mis-estimate)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, performance, slow-query]
home_hub: "60-troubleshooting"
related: ["[[database]]", "[[index-not-used]]"]
---

# Slow query — 10M+ row 의 full scan

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- p99 가 100ms → 5s 폭증, p50 은 정상
- slow query log 에 같은 query 반복
- 특정 시간대 (정산 / 통계 batch) 와 일치

## 원인 (root cause)

1. **인덱스 부재** — [[index-not-used]]
2. **JOIN 순서 잘못** — small × large 가 아닌 large × large
3. **subquery / IN (...) 가 N+1 처럼 expand**
4. **OFFSET 큰 값** — `LIMIT 10 OFFSET 100000` → 100010 row 읽고 100k 버림

## 해결

### Composite + covering index
```sql
-- 자주 쓰는 WHERE + ORDER BY 조합
CREATE INDEX idx_orders_user_created
  ON orders(user_id, created_at DESC)
  INCLUDE (status, amount);   -- Postgres covering

-- WHERE user_id = ? ORDER BY created_at DESC LIMIT 10
-- → index range scan + sort 불필요
```

### Pagination — keyset (no OFFSET)
```sql
-- Bad
SELECT * FROM orders ORDER BY id DESC LIMIT 10 OFFSET 100000;

-- Good — keyset (cursor) pagination
SELECT * FROM orders WHERE id < :lastId ORDER BY id DESC LIMIT 10;
```

```java
// Spring Data JPA + KeysetScrollPosition
ScrollPosition pos = ScrollPosition.forward(Map.of("id", lastId));
Window<Order> page = orderRepo.findFirst10ByOrderByIdDesc(pos);
```

### JOIN 최적화
```sql
-- Bad
SELECT * FROM order o
JOIN user u ON u.id = o.user_id
JOIN product p ON p.id = o.product_id
WHERE u.email LIKE '%@x.com';   -- user 부터 full scan

-- Good — filtering 먼저
SELECT * FROM (
  SELECT id FROM user WHERE email LIKE '%@x.com'   -- 작은 set
) u
JOIN order o ON o.user_id = u.id
JOIN product p ON p.id = o.product_id;
```

### IN 큰 list → JOIN
```sql
-- Bad (1000+ id)
SELECT * FROM order WHERE user_id IN (1, 2, 3, ..., 1000);

-- Good
WITH ids AS (VALUES (1), (2), ..., (1000))
SELECT o.* FROM order o JOIN ids ON o.user_id = ids.column1;
```

### Materialized view (Postgres) / Summary table
```sql
-- 일별 매출 — 실시간 집계 대신 미리 계산
CREATE MATERIALIZED VIEW order_daily AS
SELECT DATE(created_at) AS d, SUM(amount) AS total
FROM order GROUP BY DATE(created_at);

REFRESH MATERIALIZED VIEW order_daily;   -- 주기적
```

## 진단

```sql
-- 가장 느린 query top 10
-- MySQL
SELECT digest_text, count_star, sum_timer_wait/1e9 AS total_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY total_ms DESC LIMIT 10;

-- Postgres
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- EXPLAIN ANALYZE (실제 실행 + 시간)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

## 재발 방지

- **slow_query_log** prod ON (long_query_time = 1s 정도)
- **pg_stat_statements** extension 켜기
- **PR 룰**: 새 query 의 `EXPLAIN` 첨부
- **부하 테스트**: prod-like 데이터 (10M+) 로 회귀

## 관련

- [[database]] / [[index-not-used]] / [[planner-wrong-index]]
