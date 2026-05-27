---
title: "troubleshooting · Auto-increment gap"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, sequence, auto-increment]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# Auto-increment gap — id 가 1, 2, 5, 6 처럼 비어있음

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

- `SELECT id FROM order ORDER BY id` 결과 1, 2, 5, 6, 9, 10... gap
- id 가 연속이라 가정한 외부 시스템 / report 가 깨짐
- "주문 번호" 로 노출하면 사용자가 누락 의심

## 원인 (root cause)

**정상 동작**. 이유:
1. InnoDB 가 transaction 시작 시 id 예약 → rollback 해도 반환 X (성능)
2. Failover / restart 시 cache (innodb_autoinc_lock_mode=2) 가 jump
3. `INSERT ... ON DUPLICATE KEY UPDATE` 가 id 소비
4. Postgres sequence 도 nextval() 직후 rollback 하면 gap

## 해결

### "Gap 가 정상" 을 받아들이기 — 권장

- id 는 unique identifier 일뿐, 연속 의미 부여 X
- 사용자에게 노출하는 "주문 번호" 는 별도 컬럼 (e.g. `order_no VARCHAR`) 으로

```java
@Entity
public class Order {
    @Id @GeneratedValue(strategy = IDENTITY)
    private Long id;   // 내부 — gap OK

    @Column(unique = true, nullable = false)
    private String orderNo;   // 외부 노출 — UUID / Snowflake / 연속 별도 관리
}
```

### 연속 sequence 필요 시 — gap 회피 패턴

별도 counter table + lock:
```sql
CREATE TABLE order_no_counter (
    bucket DATE PRIMARY KEY,
    next_val BIGINT NOT NULL
);

-- 트랜잭션
BEGIN;
SELECT next_val FROM order_no_counter WHERE bucket = CURRENT_DATE FOR UPDATE;
UPDATE order_no_counter SET next_val = next_val + 1 WHERE bucket = CURRENT_DATE;
INSERT INTO order (order_no, ...) VALUES (CONCAT(DATE_FORMAT(NOW(),'%Y%m%d'), '-', LPAD(next_val, 6, '0')), ...);
COMMIT;
```

→ throughput 영향 (직렬화). 정말 연속이 필요한 도메인 (회계 invoice) 만.

### Lock-free 대안 — Snowflake / UUIDv7
```java
// Snowflake (timestamp + machine + sequence)
long id = snowflake.nextId();
// 시간 순서 가까운 unique id, gap 없는 distributed
```

### UUID vs sequential

| | UUID | AUTO_INCREMENT | Snowflake |
| --- | --- | --- | --- |
| 분산 | ✅ | ✗ (single master) | ✅ |
| 시간 순서 | ✗ (random) | ✅ | ✅ |
| 인덱스 효율 | ✗ (random insert) | ✅ | ✅ |
| 노출 안전 | ✅ | ✗ (count 노출) | △ |

→ public id 는 UUID / Snowflake, 내부 PK 는 AUTO_INCREMENT 패턴 흔함.

## 진단

```sql
-- 다음 id 확인
SHOW TABLE STATUS LIKE 'order';   -- MySQL Auto_increment 컬럼
SELECT last_value FROM order_id_seq;   -- Postgres

-- Gap 찾기
SELECT a.id + 1 AS gap_start, MIN(b.id) - 1 AS gap_end
FROM order a LEFT JOIN order b ON b.id > a.id
WHERE b.id IS NULL OR b.id > a.id + 1
GROUP BY a.id
HAVING gap_start <= gap_end;
```

## 관련

- [[database]]
- 책: Designing Data-Intensive Applications (ID 생성 분산)
