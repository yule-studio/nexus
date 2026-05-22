---
title: "DynamoDB 01-1 — Table + Item + Partition Key + Sort Key"
kind: knowledge
status: current
created_at: 2026-05-22T20:52:00+09:00
tags: [aws, dynamodb, fundamentals, table, item, partition-key, sort-key]
home_hub: dynamodb
---

# DynamoDB Table + Item + Key

> Table = items (rows). Each item = attributes (columns). Primary Key = PK 또는 PK+SK. Partition Key 가 partition 결정.

## 1. 구조

```
Table: bkurs-orders
├─ Item 1
│   ├─ orderId      (PK, S)  = "O-001"
│   ├─ createdAt    (SK, S)  = "2026-05-22T15:00:00Z"
│   ├─ userId       (S)      = "u-alice"
│   ├─ amount       (N)      = 50000
│   ├─ items        (L)      = [...]
│   └─ status       (S)      = "PAID"
├─ Item 2 (다른 attributes 도 가능 — schema-less)
└─ ...
```

## 2. Primary Key 2 종류

### Simple (PK only)
```
PK: userId
→ userId 하나로 unique
```

### Composite (PK + SK)
```
PK: userId (alice)
SK: orderId (O-001, O-002, ...)
→ (alice, O-001), (alice, O-002), ... 동시 가능
→ 같은 PK 의 items 가 sorted (SK 로)
```

## 3. Attribute Types

| Type | 표시 | 예 |
| --- | --- | --- |
| String | S | `"alice"` |
| Number | N | `123.45` |
| Binary | B | base64 |
| Boolean | BOOL | true / false |
| Null | NULL | null |
| List | L | `[1, "two", false]` |
| Map | M | `{"key": "value"}` |
| String Set | SS | `["a", "b", "c"]` (unique) |
| Number Set | NS | `[1, 2, 3]` |
| Binary Set | BS | `[base64, ...]` |

→ **schema-less** — 각 item 의 attribute 가 달라도 OK.

## 4. Table 생성

```bash
aws dynamodb create-table --table-name bkurs-orders \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=orderId,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=orderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/bkurs-ddb \
  --tags Key=Environment,Value=prod
```

| 옵션 | 의미 |
| --- | --- |
| `attribute-definitions` | 만 declare 한 attribute (PK / SK / Index key) |
| `key-schema` | HASH (PK) / RANGE (SK) |
| `billing-mode` | PAY_PER_REQUEST / PROVISIONED |
| `stream-specification` | Streams 활성 |
| `sse-specification` | KMS 암호화 |

## 5. Item 의 한도

| | |
| --- | --- |
| Item size | 400 KB max |
| Attribute name | 255 chars |
| Attribute value | 가변 |
| Attributes / item | unlimited |
| Sort Key 의 length | 1024 bytes |
| Partition Key 의 length | 2048 bytes |
| Item 의 size 가 400 KB 초과 시 | S3 에 큰 file 저장 + DDB 에 reference |

## 6. CRUD — Item

### PUT (insert / update)
```bash
aws dynamodb put-item --table-name bkurs-orders \
  --item '{
    "userId": {"S": "alice"},
    "orderId": {"S": "O-001"},
    "amount": {"N": "50000"},
    "status": {"S": "PAID"},
    "items": {"L": [
      {"M": {"name": {"S": "book"}, "qty": {"N": "2"}}}
    ]},
    "createdAt": {"S": "2026-05-22T15:00:00Z"}
  }'
```

### GET (single item)
```bash
aws dynamodb get-item --table-name bkurs-orders \
  --key '{"userId":{"S":"alice"},"orderId":{"S":"O-001"}}' \
  --consistent-read
```

### UPDATE
```bash
aws dynamodb update-item --table-name bkurs-orders \
  --key '{"userId":{"S":"alice"},"orderId":{"S":"O-001"}}' \
  --update-expression "SET #s = :new_status, updatedAt = :ts" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":new_status":{"S":"SHIPPED"},":ts":{"S":"2026-05-22T16:00:00Z"}}'
```

### DELETE
```bash
aws dynamodb delete-item --table-name bkurs-orders \
  --key '{"userId":{"S":"alice"},"orderId":{"S":"O-001"}}'
```

## 7. Update Expression syntax

```
SET attr = :val             — 설정
SET counter = counter + :n  — atomic counter
ADD #c :n                   — Add to Number / Set
REMOVE attr                  — 삭제
DELETE #s :v                 — Set 에서 element 제거

조건부 (ConditionExpression):
- attribute_exists(...)
- attribute_not_exists(...)
- begins_with(...)
- size(...)
- contains(...)
```

### 예
```bash
# Atomic counter
aws dynamodb update-item --table-name visits \
  --key '{"path":{"S":"/home"}}' \
  --update-expression "ADD count :one" \
  --expression-attribute-values '{":one":{"N":"1"}}'

# Conditional write (race condition 방지)
aws dynamodb update-item --table-name bkurs-orders \
  --key '{"userId":{"S":"alice"},"orderId":{"S":"O-001"}}' \
  --update-expression "SET #s = :paid" \
  --condition-expression "#s = :pending" \
  --expression-attribute-names '{"#s":"status"}' \
  --expression-attribute-values '{":pending":{"S":"PENDING"},":paid":{"S":"PAID"}}'
```

## 8. Key 설계 — Best Practice

### 1) High Cardinality PK
```
✓ userId, orderId, deviceId (unique 많음)
✗ status (PENDING/PAID/SHIPPED 만 — partition 3개)
```

→ Hot partition 회피.

### 2) Single-Table Design (★)
```
Table: bkurs-app

PK            | SK              | Attributes
--------------|-----------------|------------------
USER#alice    | PROFILE         | name, email, ...
USER#alice    | ORDER#O-001     | amount, ...
USER#alice    | ORDER#O-002     | ...
ORDER#O-001   | METADATA        | amount, ...
PRODUCT#P-100 | METADATA        | name, ...
```

→ 같은 table 안 여러 entity. Access pattern 별로 PK/SK 설계.

### 3) Composite Key Patterns
```
USER#alice    | ORDER#2026#05#001        # 연월일 sort
USER#alice    | TS#1716365400000          # timestamp sort
COMPANY#xxx   | EMPLOYEE#dept#manager#name  # hierarchical
```

## 9. Conditional Writes

```bash
# Item 이 없을 때만 (insert only)
aws dynamodb put-item --table-name users \
  --item '{"userId":{"S":"alice"},"email":{"S":"alice@bkurs.com"}}' \
  --condition-expression "attribute_not_exists(userId)"

# 이미 있으면 ConditionalCheckFailedException
```

활용:
- Distributed lock
- Idempotency
- Optimistic concurrency control (version field)

## 10. TTL (Time to Live)

```bash
aws dynamodb update-time-to-live --table-name sessions \
  --time-to-live-specification "Enabled=true,AttributeName=expireAt"

# Item 에 unix timestamp (epoch seconds)
aws dynamodb put-item --table-name sessions \
  --item '{
    "sessionId":{"S":"sess-001"},
    "userId":{"S":"alice"},
    "expireAt":{"N":"1716368400"}
  }'
```

| | |
| --- | --- |
| 자동 delete | 만료 후 0-48시간 |
| Streams 이벤트 | TTL delete = REMOVE event |
| 비용 | TTL delete 무료 (WCU 안 씀) |

## 11. PartiQL (SQL-like syntax)

```bash
aws dynamodb execute-statement \
  --statement "SELECT * FROM \"bkurs-orders\" WHERE userId = 'alice'"

aws dynamodb execute-statement \
  --statement "UPDATE \"bkurs-orders\" SET status = 'PAID' WHERE userId = 'alice' AND orderId = 'O-001'"
```

> Query / Scan 의 SQL syntax. 같은 비용. 가독성 ↑.

## 12. CDK / Terraform

```typescript
new dynamodb.Table(this, 'Orders', {
  tableName: 'bkurs-orders',
  partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
  sortKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
  billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
  encryption: dynamodb.TableEncryption.CUSTOMER_MANAGED,
  encryptionKey: kmsKey,
  pointInTimeRecovery: true,
  timeToLiveAttribute: 'expireAt',
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Low cardinality PK (status 등) | high cardinality (userId, sessionId) |
| Multi-table 관계형 설계 | Single-Table Design |
| Item size > 400 KB | S3 에 큰 file + DDB reference |
| Scan 대량 사용 | Query + GSI 설계 |
| Optimistic concurrency 없음 | ConditionExpression + version |
| TTL granularity 가정 (즉시 삭제) | 0-48h 후 |

## 14. 다음

- ▶ [[02-capacity-and-pricing]]
- ▶ [[03-consistency-and-global-tables]]
- ▶ [[../02-access-patterns/01-query-vs-scan]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [DynamoDB Core Components](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html) |
| [Single-Table Design](https://www.alexdebrie.com/posts/dynamodb-single-table/) |
| [PartiQL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ql-reference.html) |

## 16. 참고

- [[../dynamodb]]
- [[../02-access-patterns/01-query-vs-scan]]
