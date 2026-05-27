---
title: "troubleshooting · 인덱스 미사용 (full table scan)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, index, performance, explain]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# 인덱스 미사용 — `EXPLAIN type=ALL` / full scan

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- 평소 50ms 쿼리가 5s 로 폭증, table 크기 증가에 정확히 비례
- `EXPLAIN`:
  ```
  type=ALL, rows=10000000, Extra=Using where
  ```
- DataDog / pg_stat_statements 가 같은 query 의 mean_time 급증

## 원인 (root cause)

인덱스가 있어도 쓰이지 않는 패턴:

```sql
-- LIKE leading wildcard
SELECT * FROM users WHERE name LIKE '%kim';   -- 인덱스 X

-- 함수 wrap
SELECT * FROM orders WHERE DATE(created_at) = '2026-05-27';   -- 인덱스 X
SELECT * FROM users WHERE LOWER(email) = 'a@x.com';

-- 타입 mismatch (implicit cast)
SELECT * FROM users WHERE phone = 01012345678;   -- phone VARCHAR, 인자 INT

-- OR 가 두 컬럼 별도 인덱스
SELECT * FROM users WHERE name = 'kim' OR email = 'x';

-- NULL 처리 (Postgres 는 일부 인덱스 OK, MySQL B-tree 는 OK 지만 IS NULL 변환 함수 시 X)
SELECT * FROM users WHERE COALESCE(deleted_at, '...') = '...';

-- 통계 stale
ANALYZE TABLE users;  -- 안 했으면 planner 가 잘못 선택
```

## 해결

### LIKE
```sql
LIKE 'kim%'                    -- prefix match → 인덱스 OK
-- 또는 전체 텍스트 검색
CREATE FULLTEXT INDEX ft_name ON users(name);   -- MySQL
SELECT * FROM users WHERE MATCH(name) AGAINST('kim');

-- Postgres trigram
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING gin(name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%kim%';
```

### 함수 인덱스
```sql
-- MySQL 8+
CREATE INDEX idx_users_email_lower ON users((LOWER(email)));

-- Postgres
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
CREATE INDEX idx_orders_date ON orders(DATE(created_at));
```

또는 데이터 자체를 normalize (저장 시 lowercase / DATE 컬럼 분리).

### 타입 일치
```java
// JPA
@Column(name = "phone", length = 20)
private String phone;
// 절대 Long phone — 010 prefix 손실 + 타입 mismatch
```

### Composite index (OR → UNION)
```sql
SELECT * FROM users WHERE name = 'kim'
UNION
SELECT * FROM users WHERE email = 'x';
-- 각각 단일 컬럼 인덱스 사용
```

### Covering index (SELECT 컬럼 포함)
```sql
-- 인덱스가 SELECT 컬럼 다 포함 → 테이블 접근 X (Index Only Scan)
CREATE INDEX idx_orders_user_status_covering ON orders(user_id, status) INCLUDE (amount, created_at);
SELECT amount, created_at FROM orders WHERE user_id = ? AND status = 'PAID';
```

### 통계 갱신
```sql
-- MySQL
ANALYZE TABLE orders;

-- Postgres
ANALYZE orders;
-- 자동 (autovacuum)
```

## 진단

```sql
-- MySQL
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;
SHOW INDEX FROM orders;

-- Postgres
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
SELECT indexname, indexdef FROM pg_indexes WHERE tablename='orders';

-- 인덱스 미사용 통계 (Postgres)
SELECT relname, idx_scan, seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND n_live_tup > 10000
ORDER BY seq_scan DESC;
```

## 재발 방지

- **PR 룰**: 새 쿼리 추가 시 `EXPLAIN` 출력 PR body 첨부 의무
- **slow query log**: long_query_time = 1 (MySQL) / log_min_duration_statement = 1000 (Postgres)
- **테스트**: 큰 데이터 fixture (10k+) 로 query 성능 회귀 검증
- **모니터링**: pg_stat_statements / Performance Schema → 평균 응답시간 / 호출 횟수

## 관련

- [[database]] / [[../backend/n-plus-one-query]] / [[deadlock-mysql]]
