---
title: "AWS SNS — Simple Notification Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:40:00+09:00
tags:
  - aws
  - messaging
  - sns
  - pubsub
---

# AWS SNS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SNS 개념 + 사용 |

**[[messaging|↑ Messaging]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**Pub/Sub fan-out** — 1 메시지 → N 구독자. SQS / Lambda / HTTP / email / SMS / push 등 다양한 subscriber.

---

## 2. 왜

- producer 가 N consumer 일일이 호출 X
- 새 consumer 추가 시 producer 변경 X
- 자연스러운 broadcast / event-driven

대안:
- **SQS 단독** — 1:1
- **EventBridge** — event bus + filtering / scheduling
- **Kafka / MSK** — 영속 + replay + 거대 throughput

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Topic** | publish 대상 (`arn:aws:sns:...:topic-name`) |
| **Subscription** | topic 의 receiver (SQS / Lambda / email / HTTP / SMS / push) |
| **Message Filtering** | subscription 별 filter — 일부만 받음 |
| **FIFO Topic** | 순서 + 중복 제거 (subscription 도 FIFO SQS) |

---

## 4. 설치 / 사용

### 4.1 CLI

```bash
# topic
aws sns create-topic --name myapp-events

# subscribe
aws sns subscribe --topic-arn arn:aws:sns:...:myapp-events \
  --protocol sqs --notification-endpoint arn:aws:sqs:...:myqueue

aws sns subscribe --topic-arn ... --protocol lambda --notification-endpoint arn:aws:lambda:...:fn

aws sns subscribe --topic-arn ... --protocol email --notification-endpoint admin@example.com

# publish
aws sns publish --topic-arn ... --message "hello" --subject "alert"
```

### 4.2 Terraform — fan-out

```hcl
resource "aws_sns_topic" "events" {
  name = "myapp-events"
}

# 여러 SQS 구독
resource "aws_sns_topic_subscription" "to_email_queue" {
  topic_arn = aws_sns_topic.events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.email.arn
  filter_policy = jsonencode({
    event_type = ["user.signup", "user.password_reset"]
  })
}

resource "aws_sns_topic_subscription" "to_audit_queue" {
  topic_arn = aws_sns_topic.events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.audit.arn
}

# SQS policy — SNS 가 send 허용
resource "aws_sqs_queue_policy" "email" {
  queue_url = aws_sqs_queue.email.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.email.arn
      Condition = { ArnEquals = { "aws:SourceArn" = aws_sns_topic.events.arn } }
    }]
  })
}
```

### 4.3 SDK

```python
import boto3, json
sns = boto3.client("sns")

sns.publish(
    TopicArn="arn:aws:sns:...:myapp-events",
    Message=json.dumps({"event": "user.signup", "user_id": 123}),
    MessageAttributes={
        "event_type": {"DataType": "String", "StringValue": "user.signup"}
    }
)
```

`MessageAttributes` 가 **filter policy** 와 매칭.

---

## 5. Fan-out 표준 패턴

```
Producer → SNS topic
            ├── SQS Queue 1 → Worker (email)
            ├── SQS Queue 2 → Worker (audit log)
            ├── Lambda     (real-time)
            └── HTTPS      (외부 webhook)
```

→ producer 는 1번 publish, AWS 가 모든 subscriber 에 전달.

각 subscriber 가 자기 속도로 처리 (SQS queue + worker = backpressure).

---

## 6. Filtering

```hcl
filter_policy = jsonencode({
  event_type = ["user.signup", "user.deleted"]
  region     = [{ "prefix": "ap-" }]
  priority   = [{ "numeric": [">=", 100] }]
})
```

`anything-but`, `numeric`, `prefix`, `exists`, ... 다양한 연산자.

→ subscriber 별 관심 이벤트만 받음. 서버 비용 / 부하 ↓.

---

## 7. FIFO Topic

순서 + dedup 필요 시:

```hcl
resource "aws_sns_topic" "events" {
  name                        = "myapp-events.fifo"
  fifo_topic                  = true
  content_based_deduplication = true
}
```

subscribe 도 FIFO SQS 만.

---

## 8. SMS / Email / Mobile Push

| Protocol | 의미 |
| --- | --- |
| `email` / `email-json` | 알림 메일 |
| `sms` | 전화번호 (글로벌 가능) |
| `https` / `http` | webhook (외부 서비스) |
| `lambda` | Lambda 실행 |
| `sqs` | SQS 큐로 |
| `application` | 모바일 push (APNs / FCM) |

OTA alerts / on-call notification 에 자주.

---

## 9. SNS Mobile Push

```bash
# Platform application (APNs / FCM 등록)
aws sns create-platform-application --name myapp-ios \
  --platform APNS --attributes ...

# Endpoint (device token)
aws sns create-platform-endpoint --platform-application-arn ... --token "device-token"

# 발송
aws sns publish --target-arn ... --message-structure json --message '{"APNS":"..."}'
```

→ 푸시 알림 broadcast.

---

## 10. 비용 (Seoul)

```
$0.50 / 1M publish
SMS: 국가 별 ($0.005-$2/SMS)
Mobile push: $0.50 / 1M
Email: $2 / 100K
HTTPS / SQS / Lambda delivery: $0.60 / 1M
Data transfer 별도
```

저렴 — 거의 무료. SMS / Email 은 별도.

---

## 11. 사용 시나리오

- broadcast event (user signup → email + audit + analytics)
- CloudWatch Alarm → SNS → email / SMS / Slack
- S3 event → SNS → multiple consumers
- microservice integration

부적합:
- 1:1 큐 (SQS 직접)
- 영속 / replay (Kafka)
- 복잡 routing (EventBridge)

---

## 12. 함정

### 12.1 retry / DLQ
HTTP / Lambda subscription 실패 → retry → 결국 drop. **subscription 의 DLQ** 설정.

### 12.2 filter policy 의 OR/AND
같은 key 안 = OR, 다른 key = AND.

### 12.3 message 256 KB
큰 데이터 = S3 + 포인터.

### 12.4 subscription confirmation
email / HTTPS subscriber = 첫 가입 confirmation 필요.

### 12.5 unsubscribe link
email subscription 에 자동 — 누가 클릭 시 즉시 해지.

### 12.6 FIFO 의 dedup 5분
같은 dedup ID 5분 안에 publish = 무시.

### 12.7 cross-region
SNS = region 단위. cross-region 은 외부 fanout.

---

## 13. 학습 자료

- AWS SNS docs
- **SNS Workshop**

---

## 14. 관련

- [[messaging]] — Messaging hub
- [[sqs]] — fan-out 의 받는 쪽
- [[lambda]] — subscriber
- [[cloudwatch]] — Alarm → SNS
