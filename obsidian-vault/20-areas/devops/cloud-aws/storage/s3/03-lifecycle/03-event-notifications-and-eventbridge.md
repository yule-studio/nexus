---
title: "S3 03-3 — Event Notifications + EventBridge"
kind: knowledge
status: current
created_at: 2026-05-22T18:54:00+09:00
tags: [aws, s3, lifecycle, event-notification, eventbridge, lambda, sqs, sns]
home_hub: s3
---

# S3 Event Notifications + EventBridge

> Object lifecycle 변경 trigger. 4 destination (Lambda / SQS / SNS / EventBridge). Filter (prefix/suffix).

## 1. Event 종류

| Event | 의미 |
| --- | --- |
| `s3:ObjectCreated:*` | PUT / POST / COPY / CompleteMultipartUpload |
| `s3:ObjectCreated:Put` | PUT 만 |
| `s3:ObjectCreated:Post` | POST |
| `s3:ObjectCreated:Copy` | |
| `s3:ObjectCreated:CompleteMultipartUpload` | |
| `s3:ObjectRemoved:*` | delete |
| `s3:ObjectRemoved:Delete` | permanent delete |
| `s3:ObjectRemoved:DeleteMarkerCreated` | versioning 의 delete marker |
| `s3:ObjectRestore:*` | Glacier restore start/completed |
| `s3:Replication:*` | replication failed / completed / threshold |
| `s3:LifecycleTransition` | storage class 변경 |
| `s3:LifecycleExpiration:*` | expire |
| `s3:ObjectTagging:*` | tag 변경 |
| `s3:ObjectAcl:Put` | ACL 변경 |
| `s3:IntelligentTiering` | IT tier 변경 |

## 2. Destination 4

| | 권장 |
| --- | --- |
| **Lambda** | 즉시 처리 (image resize / validation) |
| **SQS** | buffer (worker pool) |
| **SNS** | fan-out (여러 subscriber) |
| **EventBridge** | rich pattern + filter / multi-target |

## 3. Lambda Destination

```bash
aws lambda add-permission --function-name image-resize \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::bkurs-uploads \
  --source-account ACCT

aws s3api put-bucket-notification-configuration --bucket bkurs-uploads \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "Id": "resize-image",
      "LambdaFunctionArn": "arn:aws:lambda:ap-northeast-2:ACCT:function:image-resize",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            { "Name": "prefix", "Value": "raw/" },
            { "Name": "suffix", "Value": ".jpg" }
          ]
        }
      }
    }]
  }'
```

자세히 → [[../../compute/lambda/02-triggers/01-async-trigger]] §2

## 4. SQS Destination

```bash
QUEUE_ARN=$(aws sqs get-queue-attributes --queue-url $QUEUE \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

# S3 가 SQS 에 publish 권한
aws sqs set-queue-attributes --queue-url $QUEUE \
  --attributes "{
    \"Policy\": \"{
      \\\"Version\\\": \\\"2012-10-17\\\",
      \\\"Statement\\\": [{
        \\\"Effect\\\": \\\"Allow\\\",
        \\\"Principal\\\": { \\\"Service\\\": \\\"s3.amazonaws.com\\\" },
        \\\"Action\\\": \\\"sqs:SendMessage\\\",
        \\\"Resource\\\": \\\"$QUEUE_ARN\\\",
        \\\"Condition\\\": {
          \\\"ArnLike\\\": { \\\"aws:SourceArn\\\": \\\"arn:aws:s3:::bkurs-uploads\\\" }
        }
      }]
    }\"
  }"

aws s3api put-bucket-notification-configuration --bucket bkurs-uploads \
  --notification-configuration "{
    \"QueueConfigurations\": [{
      \"Id\": \"queue-trigger\",
      \"QueueArn\": \"$QUEUE_ARN\",
      \"Events\": [\"s3:ObjectCreated:*\"]
    }]
  }"
```

## 5. SNS Destination

```bash
TOPIC_ARN=arn:aws:sns:...:bkurs-events

aws sns set-topic-attributes --topic-arn $TOPIC_ARN \
  --attribute-name Policy \
  --attribute-value "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [{
      \"Effect\": \"Allow\",
      \"Principal\": { \"Service\": \"s3.amazonaws.com\" },
      \"Action\": \"sns:Publish\",
      \"Resource\": \"$TOPIC_ARN\",
      \"Condition\": {
        \"ArnLike\": { \"aws:SourceArn\": \"arn:aws:s3:::bkurs-uploads\" }
      }
    }]
  }"

aws s3api put-bucket-notification-configuration --bucket bkurs-uploads \
  --notification-configuration "{
    \"TopicConfigurations\": [{
      \"Id\": \"sns-trigger\",
      \"TopicArn\": \"$TOPIC_ARN\",
      \"Events\": [\"s3:ObjectCreated:*\"]
    }]
  }"
```

## 6. EventBridge Destination ★ 권장

```bash
# bucket 의 EventBridge 활성
aws s3api put-bucket-notification-configuration --bucket bkurs-uploads \
  --notification-configuration '{
    "EventBridgeConfiguration": {}
  }'
```

→ 모든 event → EventBridge default bus.

### EventBridge rule

```bash
aws events put-rule --name bkurs-uploads-images \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": { "name": ["bkurs-uploads"] },
      "object": { "key": [{ "prefix": "raw/" }, { "suffix": ".jpg" }] }
    }
  }'

aws events put-targets --rule bkurs-uploads-images \
  --targets "Id=1,Arn=arn:lambda:...:function:resize-image"
```

## 7. EventBridge 의 강점

| | S3 Direct Notification | EventBridge |
| --- | --- | --- |
| Target 수 | 1 | 5/rule (multiple) |
| Event 종류 | 16 | 모든 S3 event + cross-service |
| Pattern matching | prefix/suffix | rich (key prefix/suffix/exact, size/etag/version) |
| Multi-account | ✗ | ✓ |
| Multi-region (Pipes) | ✗ | ✓ |
| Schema Registry | ✗ | ✓ |
| Cross-service correlation | ✗ | ✓ |

## 8. EventBridge Pipes — S3 → Lambda 전 enrich

```bash
aws pipes create-pipe --name bkurs-image-pipe \
  --source arn:aws:s3:::bkurs-uploads \
  --target arn:aws:lambda:...:function:process \
  --source-parameters '{
    "FilterCriteria": {
      "Filters": [{ "Pattern": "{\"detail\":{\"object\":{\"size\":[{\"numeric\":[\">\",1000]}]}}}" }]
    }
  }' \
  --enrichment arn:aws:lambda:...:function:enrich
```

→ Source S3 → Filter → Enrich Lambda → Target Lambda. linear pipeline.

## 9. Filter 패턴

```json
// Prefix + Suffix
"Filter": {
  "Key": {
    "FilterRules": [
      { "Name": "prefix", "Value": "raw/" },
      { "Name": "suffix", "Value": ".jpg" }
    ]
  }
}
```

EventBridge:
```json
"detail": {
  "object": {
    "key": [
      { "prefix": "raw/" },
      { "suffix": ".jpg" }
    ]
  }
}
```

## 10. CloudTrail Data Event vs S3 Event

| | S3 Event | CloudTrail Data Event |
| --- | --- | --- |
| Trigger | Lambda/SQS/SNS/EB | CloudTrail S3 log |
| 비용 | 무료 | $0.10/100K events |
| Granularity | 16 event types | 모든 API |
| Audit | ✗ | ✓ |
| Use | reactive processing | compliance audit |

## 11. Order 보장 안 됨

```
같은 key 의 PUT + DELETE 가 빠르게 일어나면
→ Lambda 가 다른 순서로 받을 수 있음
```

해결:
- `eventTime` 비교
- DynamoDB conditional write
- Versioning + version ID
- Idempotency

## 12. Lambda event 구조

```json
{
  "Records": [{
    "eventVersion": "2.1",
    "eventSource": "aws:s3",
    "awsRegion": "ap-northeast-2",
    "eventTime": "2026-05-22T17:00:00.000Z",
    "eventName": "ObjectCreated:Put",
    "s3": {
      "bucket": { "name": "bkurs-uploads", "arn": "..." },
      "object": {
        "key": "raw/cat.jpg",
        "size": 102400,
        "eTag": "abc...",
        "versionId": "..."
      }
    }
  }]
}
```

EventBridge event:
```json
{
  "version": "0",
  "id": "...",
  "detail-type": "Object Created",
  "source": "aws.s3",
  "account": "ACCT",
  "time": "2026-05-22T17:00:00Z",
  "region": "ap-northeast-2",
  "detail": {
    "version": "0",
    "bucket": { "name": "bkurs-uploads" },
    "object": {
      "key": "raw/cat.jpg",
      "size": 102400,
      "etag": "abc...",
      "version-id": "...",
      "sequencer": "..."
    },
    "request-id": "..."
  }
}
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| add-permission 누락 → Lambda 호출 X | source-arn 명시 |
| URL-encoded key (한국어 / 공백) | `urllib.parse.unquote_plus` |
| Order 보장 가정 | eventTime + version ID |
| Idempotency 없음 → 중복 부작용 | Powertools idempotency |
| 1 bucket → 1 destination 제한 (event type 별) | event 분리 또는 EventBridge |
| EventBridge 비활성 | put-bucket-notification 의 `EventBridgeConfiguration: {}` |

## 14. 다음

- ▶ [[../04-security/01-encryption-sse-kms-c-bucket-keys]]
- ▶ [[../04-security/02-object-lock-and-mfa-delete]]
- ▶ [[../practice/02-presigned-upload-pipeline]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html) |
| [EventBridge for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventBridge.html) |
| [Powertools Idempotency](https://docs.powertools.aws.dev/lambda/python/latest/utilities/idempotency/) |

## 16. 참고

- [[../s3]]
- [[../../compute/lambda/02-triggers/01-async-trigger]]
- [[../../messaging/eventbridge/eventbridge]]
