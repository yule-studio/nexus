---
title: "troubleshooting · Replication lag — read replica 가 뒤처짐"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, replication, read-replica]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# Replication lag — read replica 가 master 보다 뒤처짐

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- write 후 read 가 옛 데이터 (read-after-write 불일치)
- replication lag 시간이 spike (10초 → 30분)
- `Seconds_Behind_Master` (MySQL) / `pg_replication_slots.lag` 폭증

## 원인

1. 큰 트랜잭션 / DDL 가 single thread replay (replica 가 직렬화)
2. write spike 가 replica capacity 초과
3. 큰 batch UPDATE / DELETE
4. replica 의 hardware 가 master 보다 약함

## 해결

### Multi-threaded replication 활성화 (MySQL 5.7+)
```ini
[mysqld]
slave_parallel_workers = 8
slave_parallel_type = LOGICAL_CLOCK
slave_preserve_commit_order = ON
```

### 큰 트랜잭션 분할
```java
// Bad — 1M row 한 트랜잭션 UPDATE
@Transactional
public void migrate() { orderRepo.updateAllPrice(...); }

// Good — 배치 split
@Transactional
public void migrateBatch(Long fromId, Long toId) { ... }

for (Long batch : batches) {
    migrateBatch(batch.from, batch.to);   // 각 트랜잭션 1000 row
}
```

### Read-after-write 패턴
```java
// write 후 같은 데이터를 즉시 read 해야 하면 master 로
@Transactional
public Long create(...) {
    return repo.save(entity).getId();
}

@Transactional(readOnly = true)
@Query(value = "...", nativeQuery = true)   // hint 로 master 강제
public Entity findById(Long id) { ... }
```

Spring `@Transactional(readOnly=true)` + AbstractRoutingDataSource → readOnly=true 면 replica 로. 단 lag 이슈가 있으면 master fallback.

### lag 기반 자동 라우팅
```java
public Connection getConnection() {
    long lag = checkReplicaLag();
    if (lag < 1000) return replicaDataSource.getConnection();
    return masterDataSource.getConnection();
}
```

### Postgres logical replication
```sql
-- pgpool / pgbouncer + multiple replica round-robin
-- streaming replication 가 single thread → logical 로 parallelism
```

## 진단

```sql
-- MySQL
SHOW REPLICA STATUS\G   -- Seconds_Behind_Source
SHOW PROCESSLIST;       -- replication thread state

-- Postgres
SELECT * FROM pg_stat_replication;
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) FROM pg_stat_replication;

-- 가장 큰 트랜잭션 (lag 원인)
SELECT * FROM pg_stat_activity WHERE state != 'idle' ORDER BY xact_start;
```

## 재발 방지

- 모니터링: `Seconds_Behind_Master` / replication slot lag 알람 (10s 이상)
- 큰 batch 는 트랜잭션 분할
- DDL 는 `pt-online-schema-change` (MySQL) / `pg_repack` (Postgres) 사용 — non-blocking

## 관련

- [[database]] / [[postgres-autovacuum-bloat]] / [[slow-query]]
