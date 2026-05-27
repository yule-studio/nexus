---
title: "troubleshooting · InnoDB lock 누적 — ERROR 1206"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, innodb, lock, batch]
home_hub: "60-troubleshooting"
related: ["[[database]]", "[[../backend/deadlock-mysql]]"]
---

# InnoDB `ERROR 1206 — The total number of locks exceeds the lock table size`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

```
ERROR 1206 (HY000): The total number of locks exceeds the lock table size
```

- 큰 DELETE / UPDATE 시 발생
- bulk batch / migration 중

## 원인 (root cause)

InnoDB 가 row lock 을 `innodb_buffer_pool_size` 의 일부 영역에 보관. 큰 트랜잭션이 수백만 row 잠그면 lock memory 초과.

## 해결

### 배치 분할 (가장 권장)
```sql
-- Bad
DELETE FROM order_archive WHERE created_at < '2024-01-01';   -- 10M row

-- Good — LIMIT loop
DELETE FROM order_archive WHERE created_at < '2024-01-01' LIMIT 10000;
-- application 에서 affected=0 까지 반복
```

```java
@Transactional
public int deleteBatch(LocalDate before, int limit) {
    return jdbc.update(
        "DELETE FROM order_archive WHERE created_at < ? LIMIT ?",
        before, limit
    );
}

int total = 0;
int batch;
do {
    batch = repo.deleteBatch(date, 10_000);
    total += batch;
    log.info("deleted batch={} total={}", batch, total);
} while (batch > 0);
```

### Buffer pool 확대
```ini
[mysqld]
innodb_buffer_pool_size = 16G   # 시스템 RAM 의 70%
```

→ 메모리 여유 있을 때만. 근본 해결 X.

### Partitioning — DROP PARTITION
```sql
ALTER TABLE order_archive
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- 2024 데이터 전체 삭제 → instant + no lock
ALTER TABLE order_archive DROP PARTITION p2024;
```

### Online DDL — pt-online-schema-change
```bash
pt-online-schema-change --execute \
  --alter "ENGINE=InnoDB" \
  D=db,t=order_archive
# triggers + copy + atomic rename
```

→ lock 없이 큰 변경 처리.

## 진단

```sql
SHOW ENGINE INNODB STATUS\G   -- LOCK section
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SELECT * FROM information_schema.innodb_locks;   -- MySQL 5.7
SELECT * FROM performance_schema.data_locks;     -- MySQL 8+
```

## 재발 방지

- 대량 DML 은 항상 `LIMIT N` loop
- archive / 큰 history table 은 partitioning
- pt-online-schema-change / gh-ost 사용 표준
- 모니터링: `innodb_row_lock_time_avg` 추세

## 관련

- [[database]] / [[../backend/deadlock-mysql]] / [[replication-lag]]
