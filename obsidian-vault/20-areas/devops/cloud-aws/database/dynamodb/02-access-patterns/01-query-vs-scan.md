---
title: "DynamoDB 02-1 — Query vs Scan + Access Patterns"
kind: knowledge
status: current
created_at: 2026-05-22T21:00:00+09:00
tags: [aws, dynamodb, access-pattern, query, scan, filter]
home_hub: dynamodb
---

# DynamoDB Query vs Scan

> Query (Partition Key 기반, efficient) vs Scan (full table, expensive). Filter / Projection / Pagination.

## 1. Query vs Scan

| | Query | Scan |
| --- | --- | --- |
| 조건 | Partition Key 필수 + Sort Key 옵션 | 없음 (모든 item) |
| Performance | O(log n) — Partition lookup | O(n) — full scan |
| RCU | 결과 size 만 | 모든 evaluated item |
| Pagination | LastEvaluatedKey | 같음 |
| Filter | 후처리 (RCU 계속 차감) | 같음 |
| Index | Table / GSI / LSI | Table only |
| 권장 | 일반 access | rare admin / migration |

## 2. Query — 기본

```bash
# Single PK
aws dynamodb query --table-name bkurs-orders \
  --key-condition-expression "userId = :uid" \
  --expression-attribute-values '{":uid":{"S":"alice"}}'

# PK + SK range
aws dynamodb query --table-name bkurs-orders \
  --key-condition-expression "userId = :uid AND orderId BETWEEN :start AND :end" \
  --expression-attribute-values '{
    ":uid":{"S":"alice"},
    ":start":{"S":"O-001"},
    ":end":{"S":"O-100"}
  }'

# Sort Key prefix
aws dynamodb query --table-name bkurs-orders \
  --key-condition-expression "userId = :uid AND begins_with(orderId, :prefix)" \
  --expression-attribute-values '{":uid":{"S":"alice"},":prefix":{"S":"O-2026"}}'
```

## 3. Query 옵션

```bash
aws dynamodb query --table-name bkurs-orders \
  --key-condition-expression "userId = :uid" \
  --filter-expression "amount > :min" \
  --projection-expression "orderId, amount, #s" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{
    ":uid":{"S":"alice"},
    ":min":{"N":"10000"}
  }' \
  --scan-index-forward false \
  --limit 10 \
  --consistent-read
```

| 옵션 | 의미 |
| --- | --- |
| `filter-expression` | 결과의 server-side filter (RCU 계속 차감) |
| `projection-expression` | 반환할 attribute (network 절감) |
| `scan-index-forward` | true (asc) / false (desc) |
| `limit` | item 수 (1 MB 한도) |
| `consistent-read` | strong consistency (2x RCU) |

## 4. Filter 의 함정

```
Filter = 결과 후처리
→ matched item 만 반환
→ 그러나 모든 evaluated item 의 RCU 차감
```

### 잘못된 사용
```
Query: userId = "alice"
Filter: status = "PAID"  # alice 의 1000 items 중 10 만 PAID
→ 1000 items 모두 RCU
```

### 올바른 사용
```
GSI: status-userId-index (PK=status, SK=userId)
Query: status = "PAID" AND userId = "alice"  # 10 items 만 RCU
```

→ Filter 보다 GSI 가 효율적 (만 access pattern 이 자주 사용 시).

## 5. Pagination

```python
import boto3
ddb = boto3.client("dynamodb")

items = []
last_evaluated_key = None

while True:
    kwargs = {
        "TableName": "bkurs-orders",
        "KeyConditionExpression": "userId = :uid",
        "ExpressionAttributeValues": {":uid": {"S": "alice"}},
        "Limit": 100
    }
    if last_evaluated_key:
        kwargs["ExclusiveStartKey"] = last_evaluated_key
    
    response = ddb.query(**kwargs)
    items.extend(response["Items"])
    last_evaluated_key = response.get("LastEvaluatedKey")
    
    if not last_evaluated_key:
        break
```

| 한도 |
| --- |
| Single query: max 1 MB |
| 1 MB 초과 → `LastEvaluatedKey` 자동 |
| Filter 후 < 1 MB 이면 다음 page 도 자동 호출 필요 |

## 6. Scan — 사용 자제

```bash
aws dynamodb scan --table-name bkurs-orders \
  --filter-expression "amount > :min" \
  --expression-attribute-values '{":min":{"N":"100000"}}'
```

| | |
| --- | --- |
| 모든 item evaluate | 전체 table |
| 비용 | O(table size) — 큰 비용 |
| Throttle 위험 | hot partition |
| 권장 | 1회성 admin / migration / Inventory |

## 7. Parallel Scan

```bash
# 4 worker parallel
for segment in 0 1 2 3; do
  aws dynamodb scan --table-name bkurs-orders \
    --segment $segment --total-segments 4 &
done
```

→ 큰 table 의 scan 빠르게. 그러나 RCU 도 N배.

## 8. Query + GSI

```bash
aws dynamodb query --table-name bkurs-orders \
  --index-name status-userId-index \
  --key-condition-expression "#s = :paid" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":paid":{"S":"PAID"}}'
```

자세히 → [[02-secondary-indexes-gsi-lsi]]

## 9. Common Access Patterns

### 1) User 의 최신 N orders
```
Table: bkurs-orders (PK: userId, SK: createdAt)

Query:
  userId = "alice"
  ScanIndexForward = false
  Limit = 10
```

### 2) Date range 의 orders
```
Query:
  userId = "alice"
  createdAt BETWEEN :start AND :end
```

### 3) Status 별 orders (GSI 필요)
```
GSI: status-createdAt-index (PK: status, SK: createdAt)
Query:
  status = "PENDING"
  ScanIndexForward = true
```

### 4) Top N 의 large orders
```
GSI: status-amount-index (PK: status, SK: amount)
Query:
  status = "PAID"
  ScanIndexForward = false
  Limit = 10
```

## 10. Single-Table Design 활용

```
Table: bkurs-app
PK            | SK                  | Type
--------------|---------------------|---------
USER#alice    | PROFILE             | User
USER#alice    | ORDER#O-001         | Order
USER#alice    | ORDER#O-002         | Order
PRODUCT#P-001 | METADATA            | Product
```

Access patterns:
```
1) alice 의 profile + orders:
   Query: userId = "USER#alice"

2) alice 의 orders 만:
   Query: userId = "USER#alice" AND begins_with(SK, "ORDER#")

3) alice 의 특정 order:
   GetItem: userId = "USER#alice", SK = "ORDER#O-001"
```

## 11. PartiQL — SQL syntax

```bash
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"bkurs-orders\".\"status-userId-index\" \
               WHERE \"status\" = 'PAID' AND \"userId\" = 'alice'"
```

> PartiQL 의 select 가 query / scan 자동 (key condition 있으면 query).

## 12. 성능 측정

```python
import time
start = time.time()
response = ddb.query(...)
print(f"Time: {time.time() - start:.3f}s")
print(f"Items: {response['Count']}")
print(f"Scanned: {response.get('ScannedCount', response['Count'])}")
print(f"Consumed: {response.get('ConsumedCapacity', {})}")
```

| Metric | 의미 |
| --- | --- |
| `Count` | 반환된 items |
| `ScannedCount` | filter 전 evaluated items |
| `ConsumedCapacity.CapacityUnits` | RCU 사용 |
| `LastEvaluatedKey` | pagination cursor |

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Scan 대량 사용 | GSI / Query |
| Filter 만 (no Key Condition) → all evaluated | 적절한 GSI |
| Limit 만 (1 MB cap) | pagination |
| Consistent Read 모든 query | 필요할 때만 |
| Hot partition (low cardinality PK) | random suffix / sharding |
| Parallel Scan RCU 폭주 | 신중히 |

## 14. 다음

- ▶ [[02-secondary-indexes-gsi-lsi]]
- ▶ [[03-transactions-and-batch]]
- ▶ [[../practice/01-table-with-gsi]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Query vs Scan](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html) |
| [Best Practices for Queries](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-query-scan.html) |

## 16. 참고

- [[../dynamodb]]
- [[../01-fundamentals/01-table-item-key]]
