---
title: "DynamoDB 03-1 — Streams + Kinesis + CDC (Change Data Capture)"
kind: knowledge
status: current
created_at: 2026-05-22T21:06:00+09:00
tags: [aws, dynamodb, streams, cdc, kinesis, lambda]
home_hub: dynamodb
---

# DynamoDB Streams + CDC

> 24h change log. Lambda trigger / Kinesis Data Stream sink. Event-driven architecture 핵심.

## 1. DynamoDB Streams 활성

```bash
aws dynamodb update-table --table-name bkurs-orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

| ViewType | Image |
| --- | --- |
| `KEYS_ONLY` | PK + SK 만 |
| `NEW_IMAGE` | 변경 후 item |
| `OLD_IMAGE` | 변경 전 item |
| `NEW_AND_OLD_IMAGES` | 둘 다 |

## 2. Stream 의 record

```json
{
  "eventID": "1",
  "eventName": "INSERT",
  "eventVersion": "1.1",
  "eventSource": "aws:dynamodb",
  "awsRegion": "ap-northeast-2",
  "dynamodb": {
    "ApproximateCreationDateTime": 1716365400,
    "Keys": {
      "userId": {"S": "alice"},
      "orderId": {"S": "O-001"}
    },
    "NewImage": {
      "userId": {"S": "alice"},
      "orderId": {"S": "O-001"},
      "amount": {"N": "50000"},
      "status": {"S": "PENDING"}
    },
    "SequenceNumber": "...",
    "SizeBytes": 100,
    "StreamViewType": "NEW_AND_OLD_IMAGES"
  }
}
```

| Event Name |
| --- |
| `INSERT` |
| `MODIFY` |
| `REMOVE` (TTL delete 포함) |

## 3. Lambda Trigger

```bash
aws lambda create-event-source-mapping \
  --function-name bkurs-stream-processor \
  --event-source-arn arn:aws:dynamodb:ap-northeast-2:ACCT:table/bkurs-orders/stream/2026-05-22T15:00:00.000 \
  --starting-position LATEST \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 5 \
  --parallelization-factor 10 \
  --function-response-types ReportBatchItemFailures \
  --tumbling-window-in-seconds 60 \
  --maximum-retry-attempts 3 \
  --bisect-batch-on-function-error
```

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        event_name = record["eventName"]
        new_image = record["dynamodb"].get("NewImage", {})
        old_image = record["dynamodb"].get("OldImage", {})
        
        if event_name == "INSERT":
            send_email_confirmation(new_image)
        elif event_name == "MODIFY":
            if old_image.get("status", {}).get("S") == "PENDING" and \
               new_image.get("status", {}).get("S") == "PAID":
                send_shipping_notification(new_image)
        elif event_name == "REMOVE":
            archive_to_s3(old_image)
    
    return {"statusCode": 200}
```

자세히 → [[../../../compute/lambda/02-triggers/03-stream-poller]] §3

## 4. Streams 의 한도

| | |
| --- | --- |
| Retention | 24h (변경 불가) |
| Shard | partition 따라 자동 (max 1000) |
| Read throughput | shard 당 1000 req/s |
| Latency | < 1초 (보통) |
| Order | per item key (PK+SK level) |

## 5. Kinesis Data Stream 통합

DynamoDB → Kinesis (더 긴 retention, 더 많은 consumer):
```bash
aws dynamodb enable-kinesis-streaming-destination --table-name bkurs-orders \
  --stream-arn arn:aws:kinesis:ap-northeast-2:ACCT:stream/bkurs-orders-stream
```

| | DDB Streams | Kinesis Data Stream |
| --- | --- | --- |
| Retention | 24h | 1d-365d |
| Order | per key | per shard |
| Consumer | Lambda + 1 별도 | enhanced fan-out 多 |
| Cost | 무료 (table 의 25% 부담) | per shard·hour + per million |
| Use | 일반 trigger | analytics / long retention |

## 6. CDC (Change Data Capture) Patterns

### 1) Audit log → S3
```python
def lambda_handler(event, context):
    for record in event["Records"]:
        audit_record = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "event": record["eventName"],
            "keys": record["dynamodb"]["Keys"],
            "old": record["dynamodb"].get("OldImage"),
            "new": record["dynamodb"].get("NewImage"),
        }
        s3.put_object(
            Bucket="bkurs-audit",
            Key=f"audit/{datetime.now().date()}/{uuid.uuid4()}.json",
            Body=json.dumps(audit_record)
        )
```

### 2) Search Sync (DDB → OpenSearch)
```python
def lambda_handler(event, context):
    for record in event["Records"]:
        if record["eventName"] in ("INSERT", "MODIFY"):
            item = deserialize(record["dynamodb"]["NewImage"])
            opensearch.index(index="orders", id=item["orderId"], body=item)
        elif record["eventName"] == "REMOVE":
            order_id = record["dynamodb"]["Keys"]["orderId"]["S"]
            opensearch.delete(index="orders", id=order_id)
```

### 3) Replication to Aggregation Table
```python
def lambda_handler(event, context):
    for record in event["Records"]:
        if record["eventName"] == "INSERT":
            new = record["dynamodb"]["NewImage"]
            user_id = new["userId"]["S"]
            amount = int(new["amount"]["N"])
            
            ddb.update_item(
                TableName="user-stats",
                Key={"userId": {"S": user_id}},
                UpdateExpression="ADD totalOrders :one, totalAmount :amount",
                ExpressionAttributeValues={
                    ":one": {"N": "1"},
                    ":amount": {"N": str(amount)}
                }
            )
```

### 4) Cross-region Replication (Global Tables 의 underlying mechanism)
DDB Global Tables 가 자동 — manual 필요 없음.

## 7. Stream Processing 의 best practice

### Idempotency
```python
@idempotent_function(persistence_store=persistence, data_keyword_argument="record")
def process(record):
    # SequenceNumber 가 idempotency key
    ...
```

### Partial Failure (ReportBatchItemFailures)
```python
def lambda_handler(event, context):
    failures = []
    for record in event["Records"]:
        try:
            process(record)
        except Exception:
            failures.append({"itemIdentifier": record["dynamodb"]["SequenceNumber"]})
    return {"batchItemFailures": failures}
```

### Powertools Batch Processor
```python
from aws_lambda_powertools.utilities.batch import BatchProcessor, EventType, process_partial_response

processor = BatchProcessor(event_type=EventType.DynamoDBStreams)

def record_handler(record):
    process(record)

def lambda_handler(event, context):
    return process_partial_response(
        event=event, record_handler=record_handler,
        processor=processor, context=context
    )
```

## 8. Streams 의 보안

```json
// Lambda Execution Role
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:DescribeStream",
    "dynamodb:GetRecords",
    "dynamodb:GetShardIterator",
    "dynamodb:ListStreams"
  ],
  "Resource": "arn:aws:dynamodb:ap-northeast-2:ACCT:table/bkurs-orders/stream/*"
}
```

## 9. Streams 의 비용

```
Free quota:
- DDB Streams = table WCU 의 25% 안 무료
- 초과 시 $0.02/100K read

→ 일반 use 에는 거의 무료
```

## 10. TTL + Streams

```
TTL delete = Stream event 발생 (REMOVE)
- 사용자 의 명시 delete = REMOVE
- TTL 자동 delete = REMOVE + userIdentity = "dynamodb.amazonaws.com"
```

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        if record["eventName"] == "REMOVE":
            user_identity = record.get("userIdentity", {})
            if user_identity.get("type") == "Service" and user_identity.get("principalId") == "dynamodb.amazonaws.com":
                # TTL expired item — archive
                archive_to_s3(record["dynamodb"]["OldImage"])
```

## 11. CloudWatch metric

| Metric | 의미 |
| --- | --- |
| `OldestStreamRecord` | 가장 오래된 미처리 record (lag) |
| `ReturnedItemCount` | Stream 의 record 수 |
| `SystemErrors` | 5xx |
| Lambda 의 `IteratorAge` | Stream 처리 lag |

```bash
aws cloudwatch put-metric-alarm --alarm-name ddb-stream-lag \
  --metric-name IteratorAge --namespace AWS/Lambda \
  --statistic Maximum --period 60 --evaluation-periods 3 \
  --threshold 60000 --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=bkurs-stream-processor
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Streams 24h retention 모름 | Kinesis 통합 (long retention) |
| Idempotency 없음 → 중복 처리 | SequenceNumber 활용 |
| Partial failure 없음 → 전체 retry | ReportBatchItemFailures |
| Order 가정 (global) | per item key only |
| TTL delete 인식 안 함 | userIdentity check |
| Lambda timeout 짧음 | batch size + window 조정 |

## 13. 다음

- ▶ [[02-backup-and-pitr]]
- ▶ [[../practice/02-streams-lambda-pipeline]]
- ▶ [[../../../compute/lambda/02-triggers/03-stream-poller]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html) |
| [Kinesis 통합](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/kds.html) |
| [Powertools Batch](https://docs.powertools.aws.dev/lambda/python/latest/utilities/batch/) |

## 15. 참고

- [[../dynamodb]]
- [[../../../compute/lambda/02-triggers/03-stream-poller]]
