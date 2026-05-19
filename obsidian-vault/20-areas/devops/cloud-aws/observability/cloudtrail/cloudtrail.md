---
title: "AWS CloudTrail — API Audit Log"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:50:00+09:00
tags:
  - aws
  - observability
  - audit
  - cloudtrail
---

# AWS CloudTrail

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | CloudTrail 개념 + 사용 |

**[[observability|↑ Observability]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 안의 **모든 API 호출** 을 자동 audit log. 누가, 언제, 어디서, 무엇을, 어떤 결과.

규제 / 보안 / 디버그의 필수 도구.

---

## 2. 왜

- "누가 RDS 삭제했지?" → CloudTrail
- 규제 (SOC2, PCI, ISO27001, K-ISMS)
- 보안 이벤트 (Root login, IAM policy change)
- 잘못된 응용 / 자동화 추적

---

## 3. 종류

| Trail | 의미 |
| --- | --- |
| **Management events** | IAM / CloudFormation / 자원 생성·삭제 (기본 90일 무료) |
| **Data events** | S3 object / Lambda invoke / DynamoDB item (별도 — 비용 ↑) |
| **Insights events** | 비정상 패턴 (CloudTrail Insights, ML 기반) |
| **Organization trail** | Organizations 의 모든 account |

기본 90일은 **CloudTrail Event History** (자동). S3 / Athena / CloudWatch Logs 통합은 trail 생성 필요.

---

## 4. Trail 생성

### 4.1 Console
CloudTrail → Trails → Create trail → S3 bucket 선택.

### 4.2 Terraform

```hcl
resource "aws_s3_bucket" "trail" {
  bucket        = "myapp-cloudtrail-logs"
  force_destroy = false
}

resource "aws_s3_bucket_policy" "trail" {
  bucket = aws_s3_bucket.trail.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AWSCloudTrailAclCheck"
        Effect    = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action    = "s3:GetBucketAcl"
        Resource  = aws_s3_bucket.trail.arn
      },
      {
        Sid       = "AWSCloudTrailWrite"
        Effect    = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.trail.arn}/AWSLogs/*"
        Condition = { StringEquals = { "s3:x-amz-acl" = "bucket-owner-full-control" } }
      }
    ]
  })
}

resource "aws_cloudtrail" "main" {
  name                          = "myapp-trail"
  s3_bucket_name                = aws_s3_bucket.trail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::important-bucket/"]
    }
  }

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.trail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_logs.arn
}
```

---

## 5. 이벤트 보기

### 5.1 Console (지난 90일)
CloudTrail → Event history → 검색.

### 5.2 CLI

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance \
  --start-time 2026-05-13 \
  --end-time 2026-05-14

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=alice
```

### 5.3 S3 + Athena (장기 / SQL)

```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion STRING,
  userIdentity STRUCT<...>,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  awsRegion STRING,
  sourceIPAddress STRING,
  userAgent STRING,
  errorCode STRING,
  errorMessage STRING,
  requestParameters STRING,
  ...
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
LOCATION 's3://myapp-cloudtrail-logs/AWSLogs/.../CloudTrail/';

SELECT eventTime, userIdentity.userName, eventName, sourceIPAddress
FROM cloudtrail_logs
WHERE eventName = 'ConsoleLogin' AND errorMessage LIKE '%Failed%'
ORDER BY eventTime DESC;
```

---

## 6. 이벤트 예 (JSON)

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "alice",
    "arn": "arn:aws:iam::...:user/alice"
  },
  "eventTime": "2026-05-14T13:00:00Z",
  "eventSource": "rds.amazonaws.com",
  "eventName": "DeleteDBInstance",
  "awsRegion": "ap-northeast-2",
  "sourceIPAddress": "1.2.3.4",
  "userAgent": "aws-cli/2.15.0",
  "requestParameters": { "dBInstanceIdentifier": "prod-db" },
  "responseElements": { ... },
  "errorCode": null
}
```

→ 누가 (alice), 언제, 어디서 (1.2.3.4), 무엇을 (RDS DeleteDBInstance), 결과 (성공).

---

## 7. 보안 알람

```hcl
resource "aws_cloudwatch_log_metric_filter" "root_login" {
  log_group_name = aws_cloudwatch_log_group.trail.name
  pattern        = "{ $.userIdentity.type = \"Root\" }"
  name           = "root-login"

  metric_transformation {
    name      = "RootLoginCount"
    namespace = "Security"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "root_login" {
  alarm_name          = "root-login-detected"
  metric_name         = "RootLoginCount"
  namespace           = "Security"
  threshold           = 0
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  period              = 60
  statistic           = "Sum"
  alarm_actions       = [aws_sns_topic.security.arn]
}
```

→ root 로그인 / IAM policy 변경 / Security Group 변경 등 즉시 알람.

---

## 8. Insights — 비정상 탐지

```hcl
resource "aws_cloudtrail" "main" {
  ...
  insight_selector {
    insight_type = "ApiCallRateInsight"
  }
  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }
}
```

→ 평소 패턴 학습 후 spike 탐지. 추가 비용.

---

## 9. Lake (CloudTrail Lake)

```bash
aws cloudtrail create-event-data-store --name myapp-lake \
  --multi-region-enabled --retention-period 90
```

→ SQL 직접 query (Athena 없이). 비싸지만 편리.

---

## 10. 비용

```
Management events:  첫 trail 90일 무료
                    추가 = $2 / 100K events
Data events:        $0.10 / 100K
Insights:           $0.35 / 100K
S3 storage:         standard
Athena query:       $5 / TB scanned
Lake:               $2.50 / GB ingest + $0.10 / GB·월 + $0.005 / GB query
```

data events 가 큰 — S3 / Lambda / DynamoDB 의 모든 호출 = 어마어마.

---

## 11. 사용 시나리오

- 보안 audit (필수)
- 규제 / 인증 (SOC2, ISO)
- "누가 삭제?" 추적
- 의심 활동 (root login, 새 region 사용, IAM policy)
- 응용 / 자동화 디버그

---

## 12. 함정

### 12.1 Trail 생성 안 함
Event History (90일) 만 있음 → 장기 보관 X.

### 12.2 multi-region 누락
한 region 의 trail 만 = 다른 region 활동 안 보임. `is_multi_region_trail = true`.

### 12.3 log file validation 누락
조작 가능. `enable_log_file_validation = true`.

### 12.4 data events 비용 폭증
모든 S3 GetObject = 거대 log. 중요 bucket 만.

### 12.5 S3 bucket policy
CloudTrail service principal 누락 = 로그 못 씀. trail 생성 시 자동.

### 12.6 KMS encryption
log group / S3 의 KMS — IAM 권한 정확히.

### 12.7 retention 무관
S3 lifecycle 로 archive (Glacier) + 만료.

### 12.8 readOnly vs writeOnly events
default = `WriteOnly`. 일부 보안 의미 있는 read 누락 (GetObject 등). `All` 권장.

---

## 13. 학습 자료

- AWS CloudTrail docs
- **AWS Security Reference Architecture**
- **CIS AWS Foundations Benchmark** — 표준 정책

---

## 14. 관련

- [[observability]] — Observability hub
- [[cloudwatch]] — metric + alarm
- [[iam]] — userIdentity
- [[security]] — 보안 종합
