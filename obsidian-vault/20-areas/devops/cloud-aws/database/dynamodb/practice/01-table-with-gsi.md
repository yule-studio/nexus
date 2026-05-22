---
title: "DynamoDB 실습 1 — Table + GSI + Access Pattern Design + Auto Scaling"
kind: practice
status: current
created_at: 2026-05-22T21:12:00+09:00
tags: [aws, dynamodb, practice, gsi, access-pattern, auto-scaling]
home_hub: dynamodb
---

# 실습 1 — Table + GSI + Auto Scaling

> 목표: Single-Table Design 의 bkurs-app + 3 GSI + Conditional writes + Auto Scaling. 45분.

## 1. 요구사항 (access patterns)

```
1. User 의 profile 조회
2. User 의 모든 orders (최신 순)
3. User 의 특정 date range orders
4. Status (PENDING / PAID) 별 모든 orders (관리자)
5. Product 의 metadata 조회
6. Top 10 high-value orders (전체)
```

## 2. Single-Table Design

```
Table: bkurs-app
PK              | SK                            | Type     | Attributes
----------------|-------------------------------|----------|------------
USER#alice      | PROFILE                       | User     | email, name, phone
USER#alice      | ORDER#2026-05-22T15:00:00#O-1 | Order    | amount, status, items
USER#alice      | ORDER#2026-05-22T16:00:00#O-2 | Order    | amount, status
PRODUCT#P-100   | METADATA                      | Product  | name, price

GSI1: status-amount-index
  PK: status (PENDING, PAID, SHIPPED, CANCELLED)
  SK: amount

GSI2: product-orderId-index (Inverted)
  PK: productId
  SK: orderId
```

## 3. Table 생성

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

KEY_ID=$(aws kms create-key --description "lab ddb" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-ddb --target-key-id $KEY_ID

aws dynamodb create-table --table-name bkurs-app \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=status,AttributeType=S \
    AttributeName=amount,AttributeType=N \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes "[
    {
      \"IndexName\": \"status-amount-index\",
      \"KeySchema\": [
        { \"AttributeName\": \"status\", \"KeyType\": \"HASH\" },
        { \"AttributeName\": \"amount\", \"KeyType\": \"RANGE\" }
      ],
      \"Projection\": { \"ProjectionType\": \"ALL\" }
    }
  ]" \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification "StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES" \
  --sse-specification "Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/lab-ddb" \
  --tags Key=Environment,Value=lab Key=Backup,Value=daily

aws dynamodb wait table-exists --table-name bkurs-app

# PITR 활성
aws dynamodb update-continuous-backups --table-name bkurs-app \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```

## 4. Sample data

```python
# seed.py
import boto3, json
from datetime import datetime, timezone
from decimal import Decimal

ddb = boto3.resource("dynamodb")
table = ddb.Table("bkurs-app")

# Users
for uid in ["alice", "bob", "charlie"]:
    table.put_item(Item={
        "PK": f"USER#{uid}",
        "SK": "PROFILE",
        "type": "User",
        "email": f"{uid}@bkurs.com",
        "name": uid.capitalize(),
        "createdAt": datetime.now(timezone.utc).isoformat()
    })

# Products
for pid in range(100, 110):
    table.put_item(Item={
        "PK": f"PRODUCT#P-{pid}",
        "SK": "METADATA",
        "type": "Product",
        "name": f"Product {pid}",
        "price": Decimal(str(pid * 100))
    })

# Orders (random)
import random
statuses = ["PENDING", "PAID", "SHIPPED"]
for i in range(100):
    user = random.choice(["alice", "bob", "charlie"])
    product = random.randint(100, 109)
    amount = random.randint(5000, 100000)
    status = random.choice(statuses)
    ts = datetime.now(timezone.utc).isoformat()
    order_id = f"O-{i:04d}"
    
    table.put_item(Item={
        "PK": f"USER#{user}",
        "SK": f"ORDER#{ts}#{order_id}",
        "type": "Order",
        "orderId": order_id,
        "userId": user,
        "productId": f"P-{product}",
        "amount": Decimal(str(amount)),
        "status": status,
        "createdAt": ts
    })

print("Seeded 100 orders + 3 users + 10 products")
```

```bash
pip install boto3
python seed.py
```

## 5. Access Pattern 구현

### Pattern 1: User profile
```python
response = table.get_item(Key={"PK": "USER#alice", "SK": "PROFILE"})
profile = response["Item"]
```

### Pattern 2: User 의 모든 orders (최신 순)
```python
response = table.query(
    KeyConditionExpression="PK = :pk AND begins_with(SK, :prefix)",
    ExpressionAttributeValues={
        ":pk": "USER#alice",
        ":prefix": "ORDER#"
    },
    ScanIndexForward=False,    # 최신 순
    Limit=10
)
orders = response["Items"]
```

### Pattern 3: User 의 특정 date range
```python
response = table.query(
    KeyConditionExpression="PK = :pk AND SK BETWEEN :start AND :end",
    ExpressionAttributeValues={
        ":pk": "USER#alice",
        ":start": "ORDER#2026-05-01",
        ":end": "ORDER#2026-05-31"
    }
)
```

### Pattern 4: Status 별 orders (GSI 활용)
```python
response = table.query(
    IndexName="status-amount-index",
    KeyConditionExpression="#s = :status",
    ExpressionAttributeNames={"#s": "status"},
    ExpressionAttributeValues={":status": "PENDING"},
    ScanIndexForward=False
)
```

### Pattern 5: Product metadata
```python
response = table.get_item(Key={"PK": "PRODUCT#P-100", "SK": "METADATA"})
```

### Pattern 6: Top 10 high-value orders (GSI + Limit)
```python
# All status 의 top 10 — multi-query
results = []
for status in ["PENDING", "PAID", "SHIPPED"]:
    response = table.query(
        IndexName="status-amount-index",
        KeyConditionExpression="#s = :status",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={":status": status},
        ScanIndexForward=False,
        Limit=10
    )
    results.extend(response["Items"])

top_10 = sorted(results, key=lambda x: x["amount"], reverse=True)[:10]
```

## 6. Conditional Write — Order status transition

```python
# PENDING → PAID 만 허용
try:
    table.update_item(
        Key={"PK": "USER#alice", "SK": "ORDER#2026-05-22T15:00:00#O-1"},
        UpdateExpression="SET #s = :new",
        ConditionExpression="#s = :old",
        ExpressionAttributeNames={"#s": "status"},
        ExpressionAttributeValues={
            ":old": "PENDING",
            ":new": "PAID"
        },
        ReturnValues="UPDATED_NEW"
    )
except ddb.meta.client.exceptions.ConditionalCheckFailedException:
    print("Order not in PENDING state")
```

## 7. TransactWriteItems — Order with Balance check

```python
client = boto3.client("dynamodb")

response = client.transact_write_items(TransactItems=[
    # 1. User balance check
    {
        "ConditionCheck": {
            "TableName": "bkurs-app",
            "Key": {"PK": {"S": "USER#alice"}, "SK": {"S": "PROFILE"}},
            "ConditionExpression": "balance >= :amount",
            "ExpressionAttributeValues": {":amount": {"N": "50000"}}
        }
    },
    # 2. User balance 차감
    {
        "Update": {
            "TableName": "bkurs-app",
            "Key": {"PK": {"S": "USER#alice"}, "SK": {"S": "PROFILE"}},
            "UpdateExpression": "SET balance = balance - :amount",
            "ExpressionAttributeValues": {":amount": {"N": "50000"}}
        }
    },
    # 3. Order create
    {
        "Put": {
            "TableName": "bkurs-app",
            "Item": {
                "PK": {"S": "USER#alice"},
                "SK": {"S": "ORDER#2026-05-22T17:00:00#O-101"},
                "orderId": {"S": "O-101"},
                "amount": {"N": "50000"},
                "status": {"S": "PENDING"}
            },
            "ConditionExpression": "attribute_not_exists(PK)"
        }
    }
])
```

## 8. Auto Scaling (Provisioned mode 일 때)

```bash
# Switch to Provisioned
aws dynamodb update-table --table-name bkurs-app \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10

# WCU Auto Scaling
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/bkurs-app" \
  --scalable-dimension dynamodb:table:WriteCapacityUnits \
  --min-capacity 5 --max-capacity 1000

aws application-autoscaling put-scaling-policy \
  --policy-name wcu-target \
  --service-namespace dynamodb \
  --resource-id "table/bkurs-app" \
  --scalable-dimension dynamodb:table:WriteCapacityUnits \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBWriteCapacityUtilization"
    }
  }'

# RCU 도
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/bkurs-app" \
  --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --min-capacity 5 --max-capacity 1000

# GSI Auto Scaling 도 별도
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/bkurs-app/index/status-amount-index" \
  --scalable-dimension dynamodb:index:WriteCapacityUnits \
  --min-capacity 5 --max-capacity 100
```

## 9. CloudWatch alarm

```bash
aws cloudwatch put-metric-alarm --alarm-name lab-ddb-throttle \
  --metric-name ThrottledRequests --namespace AWS/DynamoDB \
  --statistic Sum --period 60 --evaluation-periods 5 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --dimensions Name=TableName,Value=bkurs-app

aws cloudwatch put-metric-alarm --alarm-name lab-ddb-error \
  --metric-name SystemErrors --namespace AWS/DynamoDB \
  --statistic Sum --period 60 --evaluation-periods 3 \
  --threshold 5 --comparison-operator GreaterThanThreshold \
  --dimensions Name=TableName,Value=bkurs-app
```

## 10. PITR Test

```bash
# 1. 데이터 변경
python -c "
import boto3
ddb = boto3.resource('dynamodb')
ddb.Table('bkurs-app').delete_item(Key={'PK': 'USER#alice', 'SK': 'PROFILE'})
print('Profile deleted')
"

# 2. PITR restore (실수 직전)
TIMESTAMP_BEFORE=$(date -u -v -5M +"%Y-%m-%dT%H:%M:%SZ")
aws dynamodb restore-table-to-point-in-time \
  --source-table-name bkurs-app \
  --target-table-name bkurs-app-restored \
  --restore-date-time "$TIMESTAMP_BEFORE"

aws dynamodb wait table-exists --table-name bkurs-app-restored

# 3. Verify
aws dynamodb get-item --table-name bkurs-app-restored \
  --key '{"PK":{"S":"USER#alice"},"SK":{"S":"PROFILE"}}'
```

## 11. Clean up

```bash
aws dynamodb delete-table --table-name bkurs-app
aws dynamodb delete-table --table-name bkurs-app-restored 2>/dev/null
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/lab-ddb
```

## 12. 학습 포인트

- Single-Table Design (Multi-entity table)
- PK + SK 의 composite key (User#alice + ORDER#timestamp#id)
- GSI 의 다른 access pattern 지원
- ConditionalCheckFailedException (concurrent update 보호)
- TransactWriteItems 의 ACID (balance + order create)
- Auto Scaling per-table + per-index
- PITR 의 1초 granularity restore
- Streams 활성 (다음 실습)

## 13. 다음

- ▶ [[02-streams-lambda-pipeline]]
- ▶ [[../02-access-patterns/02-secondary-indexes-gsi-lsi]]
- ▶ [[../03-operations/01-streams-and-cdc]]

## 14. 참고

- [[../dynamodb]]
- [[../02-access-patterns/01-query-vs-scan]]
- [[../02-access-patterns/02-secondary-indexes-gsi-lsi]]
- [[../02-access-patterns/03-transactions-and-batch]]
