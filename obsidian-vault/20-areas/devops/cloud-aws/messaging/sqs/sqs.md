---
title: "AWS SQS — Simple Queue Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:35:00+09:00
tags:
  - aws
  - messaging
  - sqs
---

# AWS SQS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SQS 개념 + 사용 |

**[[messaging|↑ Messaging]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **메시지 큐** — producer push, consumer pull, at-least-once.
거의 무제한 throughput + 99.999999999% durability.

---

## 2. 왜

- 응용 분리 — producer 즉시 응답, 처리는 consumer
- backpressure / 부하 평준화
- 재시도 / 실패 격리 (DLQ)
- microservice 사이 비동기

대안:
- **SNS** — pub/sub fan-out
- **EventBridge** — event bus + routing
- **Kinesis / MSK (Kafka)** — stream / 순서 보장 / 거대 throughput
- **RabbitMQ self-hosted** — 더 풍부한 routing

---

## 3. 두 종류

| | Standard | FIFO |
| --- | --- | --- |
| 순서 | best-effort | strict order |
| 중복 | at-least-once (드물게 dup) | exactly-once |
| Throughput | 무제한 | 300 msg/s (batch 3000) |
| 사용 | 일반 | 결제 / 주문 — 순서 + dedup |

→ 대부분 **Standard**. 순서·exactly-once 가 결정적이면 FIFO.

---

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Queue** | 메시지 buffer (URL 로 식별) |
| **Message** | 최대 256 KB (S3 + 포인터로 더 큼) |
| **Visibility Timeout** | consumer 가 잡은 후 다른 consumer 안 봄 — 기본 30s |
| **Retention** | 메시지 유지 (1분-14일) |
| **DLQ** (Dead-Letter Queue) | N회 실패 메시지 격리 |
| **Long Polling** | 빈 큐 대기 (0-20s) — API call 절약 |
| **Batch** | 최대 10 메시지 |

---

## 5. 설치 / 사용

### 5.1 Queue 생성

```bash
aws sqs create-queue --queue-name myapp-jobs \
  --attributes '{
    "VisibilityTimeout":"60",
    "MessageRetentionPeriod":"345600",
    "ReceiveMessageWaitTimeSeconds":"20"
  }'
```

### 5.2 Terraform

```hcl
resource "aws_sqs_queue" "jobs" {
  name                       = "myapp-jobs"
  visibility_timeout_seconds = 60
  message_retention_seconds  = 345600         # 4일
  receive_wait_time_seconds  = 20             # long polling
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 5
  })
}

resource "aws_sqs_queue" "dlq" {
  name = "myapp-jobs-dlq"
  message_retention_seconds = 1209600         # 14일
}
```

### 5.3 SDK (Python)

```python
import boto3, json
sqs = boto3.client("sqs")
QUEUE_URL = "https://sqs.ap-northeast-2.amazonaws.com/.../myapp-jobs"

# send
sqs.send_message(QueueUrl=QUEUE_URL, MessageBody=json.dumps({"task":"send_email"}))

# batch
sqs.send_message_batch(QueueUrl=QUEUE_URL, Entries=[
    {"Id":"1", "MessageBody": ...},
    {"Id":"2", "MessageBody": ...},
])

# consume (long polling)
while True:
    resp = sqs.receive_message(
        QueueUrl=QUEUE_URL,
        MaxNumberOfMessages=10,
        WaitTimeSeconds=20,
        VisibilityTimeout=60,
    )
    for msg in resp.get("Messages", []):
        try:
            data = json.loads(msg["Body"])
            handle(data)
            sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg["ReceiptHandle"])
        except Exception:
            # 처리 안 함 → visibility timeout 후 다시 / DLQ
            pass
```

### 5.4 Lambda 통합

```hcl
resource "aws_lambda_event_source_mapping" "worker" {
  event_source_arn = aws_sqs_queue.jobs.arn
  function_name    = aws_lambda_function.worker.arn
  batch_size       = 10
  maximum_batching_window_in_seconds = 5
}
```

→ SQS → Lambda 자동 polling. 실패 시 visibility timeout 후 재시도.

---

## 6. DLQ 패턴

```
main queue → consume N회 실패 → DLQ
DLQ 알람 (CloudWatch) → 개발자 / 자동 알람
```

```hcl
redrive_policy = jsonencode({
  deadLetterTargetArn = aws_sqs_queue.dlq.arn
  maxReceiveCount     = 5
})
```

`maxReceiveCount = 5` → 5번 시도 후 DLQ.

DLQ replay:
```bash
aws sqs start-message-move-task --source-arn <DLQ_ARN> --destination-arn <MAIN_ARN>
```

---

## 7. FIFO 의 추가 옵션

```python
sqs.send_message(
    QueueUrl=FIFO_URL,
    MessageBody="...",
    MessageGroupId="order-123",        # 같은 group = 순서 보장
    MessageDeduplicationId="unique-id" # 5분 dedup
)
```

`MessageGroupId` — 같은 group 끼리 순서 보장 + 1 consumer 만 동시 처리.

---

## 8. Visibility Timeout

```
consume → 메시지 잡힘 → 다른 consumer 에 보이지 않음 (timeout 동안)
처리 완료 → delete
처리 실패 / 늦음 → timeout 후 다시 visible → 재시도
```

→ 처리 시간 보다 짧게 = 중복 / 길게 = 실패 후 재시도 늦음.

`ChangeMessageVisibility` — 처리 중 timeout 연장.

---

## 9. 비용 (Seoul)

```
$0.40 / 1M API request
무료: 1M / month
```

FIFO 는 약간 더 비쌈 ($0.50/M). data transfer 별도.

매우 저렴. 큰 throughput 도 보통 monthly $1-100.

---

## 10. 사용 시나리오

- 작업 큐 (email / image processing)
- 응용 분리 (web → worker)
- 부하 평준화 (request spike)
- S3 → SQS → Lambda
- SNS → SQS fan-out
- microservice 비동기

부적합:
- 실시간 streaming (Kinesis)
- 거대 throughput + 영속 replay (Kafka / MSK)
- 복잡 routing (EventBridge / RabbitMQ)

---

## 11. 함정

### 11.1 at-least-once 의 중복
exactly-once 가정 X — idempotent 처리.

### 11.2 message 256 KB 한계
큰 데이터 = S3 + 포인터 (Extended Client Library).

### 11.3 polling 잘못
short polling (default 0s) → API 호출 폭증. **long polling 20s** 권장.

### 11.4 visibility timeout 짧음
처리 시간 > timeout → 중복. 측정 후 설정.

### 11.5 batch delete 누락
delete 실패 = 다시 처리. batch.

### 11.6 dead letter 회수
DLQ 보고만 — 원인 분석 + replay.

### 11.7 FIFO throughput 한계
300 msg/s. high throughput FIFO 옵션 (3000/s) 별도.

---

## 12. 학습 자료

- AWS SQS docs
- **AWS Messaging** workshop

---

## 13. 관련

- [[messaging]] — Messaging hub
- [[sns]] — fan-out
- [[lambda]] — consumer
- [[mqtt-amqp-kafka|↗ MQ 이론]]
