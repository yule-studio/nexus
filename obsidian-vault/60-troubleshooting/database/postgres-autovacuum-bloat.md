---
title: "troubleshooting · Postgres autovacuum 안 돌아 table bloat"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, postgres, vacuum, mvcc, bloat]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# Postgres autovacuum bloat — dead tuple 누적으로 table 폭증

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- Table 의 row 수는 100k 인데 disk 크기 50GB
- `SELECT` 가 느려짐 (dead tuple 도 read)
- autovacuum 이 돌지 않거나 너무 늦게

## 원인 (root cause)

Postgres MVCC — UPDATE 는 새 row 추가 + 옛 row 를 dead 표시 (즉시 삭제 X). DELETE 도 dead 표시. autovacuum 이 dead tuple 정리. 임계치 못 넘으면 안 돔:

```
autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × n_live_tup
default: 50 + 0.2 × n_live_tup
```

→ 100M row 면 20M dead 누적되어야 시작. 그 사이 bloat.

## 해결

### autovacuum 공격적 설정 (table-level)
```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05,    -- 5% dead 시
  autovacuum_vacuum_threshold = 1000,
  autovacuum_analyze_scale_factor = 0.05,
  autovacuum_vacuum_cost_limit = 2000        -- vacuum 속도 ↑
);
```

### Manual VACUUM
```sql
VACUUM (VERBOSE, ANALYZE) orders;     -- non-blocking
VACUUM FULL orders;                    -- exclusive lock — peak 시 X
pg_repack -t orders                    -- non-blocking full vacuum (extension)
```

### Long-running transaction 차단 — 가장 흔한 원인
```sql
SELECT pid, age(clock_timestamp(), xact_start) AS age, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND xact_start IS NOT NULL
ORDER BY xact_start;
-- 오래된 트랜잭션이 dead tuple cleanup 막음 (visibility horizon)

-- 강제 종료 (위험)
SELECT pg_terminate_backend(<pid>);
```

→ app 측 트랜잭션이 30분 이상 열려있으면 vacuum 가 안 됨. [[../backend/hikari-connection-leak]] 참조.

## 진단

```sql
-- Bloat top 10
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0), 2) AS dead_ratio,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;

-- Table size
SELECT pg_size_pretty(pg_total_relation_size('orders'));
SELECT pg_size_pretty(pg_relation_size('orders'));   -- 데이터만 (인덱스 제외)

-- pgstattuple extension
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('orders');
```

## 재발 방지

- **모든 큰 table** 에 공격적 autovacuum 설정
- **long transaction 모니터링**:
  ```
  - alert: LongRunningTransaction
    expr: pg_stat_activity_max_tx_duration > 600
  ```
- **autovacuum log** 확인 (`log_autovacuum_min_duration = 0`)

## 관련

- [[database]] / [[replication-lag]] / [[../backend/hikari-connection-leak]]
