---
title: "troubleshooting · Foreign key constraint fails"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, postgres, foreign-key, constraint]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# Foreign key constraint fails

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

```
-- INSERT 시
ERROR 1452: Cannot add or update a child row:
  a foreign key constraint fails (`db`.`order_item`,
  CONSTRAINT `fk_order_item_order` FOREIGN KEY (`order_id`) REFERENCES `order` (`id`))

-- DELETE 시
ERROR 1451: Cannot delete or update a parent row:
  a foreign key constraint fails
```

## 원인

- INSERT child 시 parent 없음
- DELETE parent 시 child 존재
- 트랜잭션 순서 잘못 / cascade 미설정

## 해결

### Order 보장
```java
@Transactional
public void create(...) {
    Order order = orderRepo.save(new Order(...));   // 1
    OrderItem item = new OrderItem(order, ...);     // 2
    orderItemRepo.save(item);                       // FK OK
}
```

JPA `cascade = CascadeType.PERSIST` 로 자동:
```java
@OneToMany(mappedBy = "order", cascade = {PERSIST, MERGE}, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

### Cascade DELETE (의도 명시)
```sql
ALTER TABLE order_item
  ADD CONSTRAINT fk_order_item_order
  FOREIGN KEY (order_id) REFERENCES `order`(id)
  ON DELETE CASCADE;   -- parent 삭제 시 child 자동 삭제
```

또는 SET NULL / RESTRICT 선택.

### Soft delete 채택 — FK 유지하면서 "삭제"
```java
@Entity
@SQLDelete(sql = "UPDATE order SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class Order {
    LocalDateTime deletedAt;
}
```

→ 실제 row 는 살아있어 FK 안전. 단 모든 조회에 `deleted_at IS NULL` 필요.

### Deferred constraint (Postgres)
```sql
ALTER TABLE order_item
  ADD CONSTRAINT fk_order_item_order
  FOREIGN KEY (order_id) REFERENCES order(id)
  DEFERRABLE INITIALLY DEFERRED;
-- 트랜잭션 commit 시점에만 검증 → 순서 자유
```

## 데이터 정합성 함정

FK 가 없으면 (성능 위해 제거) orphan row 발생 가능:
```sql
SELECT * FROM order_item WHERE order_id NOT IN (SELECT id FROM order);
```

→ FK 가 보통 옳음. 정말 필요 시만 제거 (sharding / 분산 DB).

## 진단

```sql
-- MySQL FK 목록
SELECT * FROM information_schema.referential_constraints
WHERE constraint_schema = 'db';

-- Postgres FK
SELECT conname, conrelid::regclass AS table, pg_get_constraintdef(oid)
FROM pg_constraint WHERE contype = 'f';

-- orphan row 찾기
SELECT child.id FROM order_item child
LEFT JOIN order parent ON parent.id = child.order_id
WHERE parent.id IS NULL;
```

## 관련

- [[database]] / [[../backend/circular-dependency]]
