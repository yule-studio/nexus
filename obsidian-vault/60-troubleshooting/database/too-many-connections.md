---
title: "troubleshooting · too many connections"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, connection-pool, pgbouncer]
home_hub: "60-troubleshooting"
related: ["[[database]]", "[[../backend/hikari-connection-leak]]"]
---

# Too many connections — `FATAL: sorry, too many clients already`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

```
-- MySQL
ERROR 1040 (HY000): Too many connections

-- Postgres
FATAL: sorry, too many clients already
```

- 새 deploy 후 발생 (pod 수 증가)
- 또는 client app 의 pool 가 너무 큼

## 원인 (root cause)

`max_connections` 한도 도달. Postgres 의 경우 각 connection 이 backend process — 메모리 (~10MB) 도 소비. 무한 확대 X.

```
client app pool × pod 수 ≤ DB max_connections × 0.7 (여유)
ex) Hikari pool 20 × pod 50 = 1000 → DB max=1500
```

## 해결

### Connection pooler 도입 — pgBouncer (Postgres)

```ini
# pgbouncer.ini
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
pool_mode = transaction              # transaction-level pool
max_client_conn = 5000
default_pool_size = 50               # backend connection 50 만 사용
```

app → pgBouncer (6432) → Postgres (5432). app pool 5000 → DB 실제 connection 50.

MySQL: ProxySQL / 자체 pool.

### Hikari pool 적절히
```yaml
spring.datasource.hikari:
  maximum-pool-size: 20      # 너무 크면 DB 압박, 너무 작으면 처리량 ↓
```

→ [[../backend/hikari-connection-leak]] sizing 가이드 참조 (`(core_count × 2) + spindle`).

### DB max_connections 확대 (메모리 여유 시)

```sql
-- Postgres
ALTER SYSTEM SET max_connections = 500;
SELECT pg_reload_conf();

-- MySQL (config 변경 + 재시작)
[mysqld]
max_connections = 500
```

→ 메모리 사용량 검토. Postgres 500 connection ≈ 5GB.

### Connection leak 차단

[[../backend/hikari-connection-leak]] — `leak-detection-threshold` + try-with-resources.

## 진단

```sql
-- MySQL
SHOW PROCESSLIST;
SELECT user, host, COUNT(*) FROM information_schema.processlist GROUP BY user, host;

-- Postgres
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
SELECT pid, usename, application_name, state, query_start
FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start;
```

## 재발 방지

- pgBouncer / ProxySQL 표준 도입
- `max_connections` 산정: `client_pool × instances × 1.2 (여유)`
- 모니터링: `pg_stat_database.numbackends` / `mysql_global_status_threads_connected`

## 관련

- [[database]] / [[../backend/hikari-connection-leak]]
