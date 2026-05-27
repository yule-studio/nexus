---
title: "DynamoDB 02-3 — Transactions + BatchWriteItem + BatchGetItem"
kind: knowledge
status: current
created_at: 2026-05-22T21:04:00+09:00
tags: [aws, dynamodb, access-pattern, transactions, batch, acid]
home_hub: dynamodb
---

# DynamoDB Transactions + Batch

> Transactions (ACID across multiple items / tables, 100 items / 4 MB max). Batch (25 write / 100 read, parallel).

## 1. 4 종류 비교

| Operation | Max items | ACID | RCU/WCU | 권장 |
| --- | --- | --- | --- | --- |
| **Single Item** (PutItem/GetItem) | 1 | n/a | normal | 표준 |
| **BatchGetItem** | 100 (16 MB) | ✗ | normal | parallel read |
| **BatchWriteItem** | 25 (16 MB) | ✗ (개별 fail 가능) | normal | parallel write |
| **TransactGetItems** | 100 (4 MB) | ✓ | 2x | atomic read |
| **TransactWriteItems** | 100 (4 MB) | ✓ | 2x | atomic write |

## 2. BatchGetItem

```bash
aws dynamodb batch-get-item --request-items '{
  "users": {
    "Keys": [
      {"userId": {"S": "alice"}},
      {"userId": {"S": "bob"}},
      {"userId": {"S": "charlie"}}
    ],
    "ProjectionExpression": "userId, email"
  },
  "orders": {
    "Keys": [
      {"orderId": {"S": "O-001"}},
      {"orderId": {"S": "O-002"}}
    ]
  }
}'
```

| 한도 | |
| --- | --- |
| 100 items / request | |
| 16 MB total | |
| Parallel | server 측 |
| `UnprocessedKeys` | retry |

## 3. BatchWriteItem

```bash
aws dynamodb batch-write-item --request-items '{
  "users": [
    {"PutRequest": {"Item": {"userId": {"S": "alice"}, "name": {"S": "Alice"}}}},
    {"PutRequest": {"Item": {"userId": {"S": "bob"}, "name": {"S": "Bob"}}}},
    {"DeleteRequest": {"Key": {"userId": {"S": "old-user"}}}}
  ]
}'
```

| 한도 | |
| --- | --- |
| 25 items / request | |
| 16 MB total | |
| Operations: PutRequest / DeleteRequest | |
| 개별 fail OK | `UnprocessedItems` |
| ACID ✗ | partial success |

### Retry 패턴

```python
import boto3
ddb = boto3.client("dynamodb")

def batch_write_with_retry(items, max_retries=10):
    request_items = {"users": [{"PutRequest": {"Item": item}} for item in items]}
    
    for attempt in range(max_retries):
        response = ddb.batch_write_item(RequestItems=request_items)
        unprocessed = response.get("UnprocessedItems", {})
        if not unprocessed:
            return
        request_items = unprocessed
        time.sleep(2 ** attempt * 0.1)   # exponential backoff
    
    raise Exception(f"Failed after {max_retries} attempts")
```

## 4. TransactWriteItems — ACID

```bash
aws dynamodb transact-write-items --transact-items '[
  {
    "ConditionCheck": {
      "TableName": "users",
      "Key": {"userId": {"S": "alice"}},
      "ConditionExpression": "balance >= :amount",
      "ExpressionAttributeValues": {":amount": {"N": "50000"}}
    }
  },
  {
    "Update": {
      "TableName": "users",
      "Key": {"userId": {"S": "alice"}},
      "UpdateExpression": "SET balance = balance - :amount",
      "ExpressionAttributeValues": {":amount": {"N": "50000"}}
    }
  },
  {
    "Update": {
      "TableName": "users",
      "Key": {"userId": {"S": "bob"}},
      "UpdateExpression": "SET balance = balance + :amount",
      "ExpressionAttributeValues": {":amount": {"N": "50000"}}
    }
  },
  {
    "Put": {
      "TableName": "transfers",
      "Item": {
        "transferId": {"S": "T-001"},
        "from": {"S": "alice"},
        "to": {"S": "bob"},
        "amount": {"N": "50000"},
        "createdAt": {"S": "2026-05-22T15:00:00Z"}
      },
      "ConditionExpression": "attribute_not_exists(transferId)"
    }
  }
]'
```

### 4 operation types

| | 의미 |
| --- | --- |
| `ConditionCheck` | item 조건 검증 (write X) |
| `Put` | item 추가 |
| `Update` | item 수정 |
| `Delete` | item 삭제 |

### Atomicity
```
모든 operation 성공 → commit
하나라도 실패 → rollback (모두 적용 안 됨)
```

| 한도 | |
| --- | --- |
| 100 items / transaction | |
| 4 MB total | |
| Cross-table | ✓ |
| Cross-region (Global Tables) | ✗ |
| RCU/WCU | 2x normal |

## 5. Optimistic Concurrency Control

```python
def update_with_version(item_id, new_data, expected_version):
    try:
        ddb.update_item(
            Key={"id": {"S": item_id}},
            UpdateExpression="SET data = :data, version = version + :one",
            ConditionExpression="version = :v",
            ExpressionAttributeValues={
                ":data": {"S": new_data},
                ":one": {"N": "1"},
                ":v": {"N": str(expected_version)}
            }
        )
    except ddb.exceptions.ConditionalCheckFailedException:
        # Another writer 이 먼저 변경했음 → re-fetch + retry
        pass
```

## 6. Idempotency

### Idempotency Key
```python
def idempotent_put(key, item):
    try:
        ddb.put_item(
            TableName="orders",
            Item={**item, "idempotencyKey": {"S": key}},
            ConditionExpression="attribute_not_exists(idempotencyKey)"
        )
    except ddb.exceptions.ConditionalCheckFailedException:
        # 이미 처리됨 — silently ignore
        pass
```

### Powertools Idempotency
```python
from aws_lambda_powertools.utilities.idempotency import idempotent, DynamoDBPersistenceLayer

persistence = DynamoDBPersistenceLayer(table_name="idempotency")

@idempotent(persistence_store=persistence)
def lambda_handler(event, context):
    process_order(event)
```

## 7. Distributed Lock

```python
def acquire_lock(resource_id, owner, ttl_seconds=60):
    try:
        ddb.put_item(
            TableName="locks",
            Item={
                "resource": {"S": resource_id},
                "owner": {"S": owner},
                "expireAt": {"N": str(int(time.time()) + ttl_seconds)}
            },
            ConditionExpression="attribute_not_exists(#r) OR expireAt < :now",
            ExpressionAttributeNames={"#r": "resource"},
            ExpressionAttributeValues={":now": {"N": str(int(time.time()))}}
        )
        return True
    except ddb.exceptions.ConditionalCheckFailedException:
        return False

def release_lock(resource_id, owner):
    try:
        ddb.delete_item(
            TableName="locks",
            Key={"resource": {"S": resource_id}},
            ConditionExpression="#o = :owner",
            ExpressionAttributeNames={"#o": "owner"},
            ExpressionAttributeValues={":owner": {"S": owner}}
        )
    except:
        pass
```

## 8. Atomic Counter

```bash
aws dynamodb update-item --table-name visits \
  --key '{"path":{"S":"/home"}}' \
  --update-expression "ADD #c :one" \
  --expression-attribute-names '{"#c":"count"}' \
  --expression-attribute-values '{":one":{"N":"1"}}'
```

→ no race condition. 항상 atomic increment.

## 9. Transaction 의 비용

```
TransactWriteItems:
  - 100 items × 1 KB = 200 WCU (2x normal)
  - Cost = 200 × $0.000571 / 3600 = $0.0000317

vs Sequential UpdateItem:
  - 100 items × 1 KB = 100 WCU (no transaction)
  - Cost = $0.0000159

→ ACID 의 cost = 2x. 필요할 때만.
```

## 10. Batch vs Transaction 결정

```mermaid
flowchart TD
  Q1{Partial failure OK?}
  Q1 -->|Yes| BATCH[Batch]
  Q1 -->|No (ACID)| TXN[Transaction]
  Q2{Single item?}
  Q2 -->|Yes| SINGLE[PutItem/UpdateItem]
  Q2 -->|No| Q1
```

| Use case | 권장 |
| --- | --- |
| 100 user metadata 한번 update | Batch (partial fail OK) |
| Payment + Order create | Transaction (ACID) |
| Inventory deduct + Order | Transaction |
| Bulk import | Batch + retry |
| Counter | Single Update (atomic) |

## 11. CloudWatch metric

```
SuccessfulRequestLatency (BatchWriteItem)
TransactionConflict (race condition)
ConditionalCheckFailedRequests
ReturnedItemCount (Batch)
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| BatchWriteItem 25 limit 모름 | chunking |
| UnprocessedKeys retry 안 함 | exponential backoff |
| Transaction 의 2x WCU 모름 | 필요할 때만 |
| Cross-region transaction | same region 만 |
| Conditional write 없음 → race | ConditionExpression |
| Lock 의 TTL 없음 → deadlock | expireAt + TTL |

## 13. 다음

- ▶ [[../03-operations/01-streams-and-cdc]]
- ▶ [[../03-operations/02-backup-and-pitr]]
- ▶ [[../practice/01-table-with-gsi]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [Transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transactions.html) |
| [Batch Operations](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/batch-operation-document-api.html) |
| [Powertools Idempotency](https://docs.powertools.aws.dev/lambda/python/latest/utilities/idempotency/) |

## 15. 참고

- [[../dynamodb]]
- [[01-query-vs-scan]]
- [[02-secondary-indexes-gsi-lsi]]
