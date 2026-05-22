---
title: "S3 04-3 — Access Analyzer + CloudTrail Data Event"
kind: knowledge
status: current
created_at: 2026-05-22T19:00:00+09:00
tags: [aws, s3, security, access-analyzer, cloudtrail, macie]
home_hub: s3
---

# S3 Access Analyzer + CloudTrail Data Event

> IAM Access Analyzer = public/cross-account 자동 탐지 + CloudTrail Data Event = 모든 object access audit + Macie = 민감 정보 발견.

## 1. IAM Access Analyzer

```bash
aws accessanalyzer create-analyzer --analyzer-name bkurs-account \
  --type ACCOUNT

# Organization-wide
aws accessanalyzer create-analyzer --analyzer-name bkurs-org \
  --type ORGANIZATION
```

자동 탐지:
- Public bucket
- Cross-account bucket
- 누구나 접근 가능한 IAM Role
- public S3 ACL

## 2. Finding 예

```bash
aws accessanalyzer list-findings --analyzer-arn arn:... \
  --filter "resourceType={\"eq\":[\"AWS::S3::Bucket\"]}" \
  --query 'findings[].{resource:resource,principals:principal,public:isPublic,status:status}'
```

```json
{
  "resource": "arn:aws:s3:::bkurs-uploads",
  "principal": { "AWS": ["*"] },
  "isPublic": true,
  "status": "ACTIVE"
}
```

## 3. EventBridge — Finding 알림

```yaml
EventPattern:
  source: [aws.access-analyzer]
  detail-type: ["Access Analyzer Finding"]
  detail:
    status: [ACTIVE]
    isPublic: [true]
Targets:
  - Arn: arn:lambda:...:function:notify-security
```

## 4. 자동 remediation

```python
# Lambda
import boto3
def lambda_handler(event, context):
    bucket = event["detail"]["resource"].split(":")[-1]
    s3 = boto3.client("s3")
    s3.put_public_access_block(Bucket=bucket,
        PublicAccessBlockConfiguration={
            "BlockPublicAcls": True,
            "IgnorePublicAcls": True,
            "BlockPublicPolicy": True,
            "RestrictPublicBuckets": True
        })
```

## 5. CloudTrail Management Event (default)

자동 — 모든 bucket level API:
```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutBucketPolicy \
  --max-items 20

# Resource 별
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=bkurs-uploads \
  --max-items 30
```

| Logged | Not Logged |
| --- | --- |
| Bucket 생성/삭제 | object GET/PUT |
| Policy / Encryption / Lifecycle 변경 | object content access |
| Lock / Versioning 변경 | |
| Replication 설정 | |
| Access Point 생성 | |

## 6. CloudTrail Data Event

object level access (GET/PUT/DELETE):
```bash
aws cloudtrail put-event-selectors --trail-name bkurs-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::bkurs-audit/"]
    }]
  }]'
```

| 비용 |
| --- |
| $0.10 / 100K Data Events |

> 모든 bucket activate 시 비용 폭주. **민감 bucket 만**.

## 7. Data Event 의 finding

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=audit.log \
  --max-items 30
```

→ 누가 / 언제 / 어디서 audit.log 를 GET 했는지.

## 8. CloudTrail Lake — query

```bash
aws cloudtrail start-query \
  --query-statement "SELECT eventTime, userIdentity.principalId, eventName \
                     FROM \$STORE_ID \
                     WHERE resources[0].arn LIKE 'arn:aws:s3:::bkurs-audit/%' \
                     AND eventTime > '2026-05-22T00:00:00Z' \
                     ORDER BY eventTime DESC LIMIT 100"
```

## 9. S3 Server Access Logging

```bash
aws s3api put-bucket-logging --bucket bkurs-uploads \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "bkurs-logs",
      "TargetPrefix": "access-logs/bkurs-uploads/"
    }
  }'
```

| | Server Access Logging | CloudTrail Data Event |
| --- | --- | --- |
| 비용 | 저장 비용만 ($0.025/GB) | $0.10/100K |
| Format | Apache style | JSON |
| Latency | 수시간 | 분 |
| Detail | request info | API call info |
| 권장 | bulk audit / forensic | 실시간 alert |

## 10. Amazon Macie

```bash
aws macie2 enable-macie

aws macie2 create-classification-job --job-type ONE_TIME \
  --s3-job-definition '{
    "bucketDefinitions": [{ "accountId": "ACCT", "buckets": ["bkurs-uploads"] }]
  }' \
  --name "scan-bkurs-uploads"
```

자동 분류:
- PII (이름, 주민번호, 신용카드, 이메일)
- Credentials (API key, password, secret)
- 한국 주민등록번호, 전화번호
- AWS Access Key
- DB connection string

발견 시:
- EventBridge
- Security Hub finding

## 11. Trusted Advisor

```bash
aws support describe-trusted-advisor-check-result \
  --check-id "Pfx0RwqBli"   # S3 Bucket Permissions
```

→ public bucket / large bucket / 비용 관련 자동 검사.

## 12. AWS Config rule

자동 compliance:
```bash
# S3 bucket public read prohibited
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-public-read-prohibited",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}'

# S3 default encryption
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-server-side-encryption-enabled",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }
}'

# S3 versioning
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-versioning-enabled",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "S3_BUCKET_VERSIONING_ENABLED"
  }
}'
```

→ noncompliant bucket 알림 + remediation (Lambda).

## 13. Security Hub 통합

```bash
aws securityhub enable-security-hub --enable-default-standards
```

→ Access Analyzer / Macie / Config / GuardDuty / Inspector 의 발견 unified.

## 14. 권장 baseline 체크리스트

```
[ ] Account-level Block Public Access ENABLED
[ ] IAM Access Analyzer 활성
[ ] CloudTrail Management Event 활성
[ ] CloudTrail Data Event (sensitive bucket 만)
[ ] AWS Config rule 4종 (public/encryption/versioning/logging)
[ ] S3 Server Access Logging (audit bucket)
[ ] Default Encryption SSE-KMS + Bucket Keys
[ ] Object Lock (compliance / audit)
[ ] Replication (DR)
[ ] Versioning + Lifecycle
[ ] Macie (PII bucket)
[ ] Security Hub
[ ] EventBridge → Lambda 자동 remediation
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Access Analyzer 비활성 | 활성 |
| CloudTrail Data Event 모든 bucket | sensitive 만 |
| Server Access Logging cycle (own bucket) | 다른 bucket |
| Macie 비용 인지 못 함 | per-GB scan |
| Config rule 없음 | 표준 4종 권장 |
| Trusted Advisor 안 봄 | weekly review |

## 16. 다음

- ▶ [[../practice/01-static-site-cloudfront]]
- ▶ [[../practice/02-presigned-upload-pipeline]]
- ▶ [[../practice/03-lifecycle-glacier-deep-archive]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) |
| [CloudTrail S3 Data Event](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cloudtrail-logging.html) |
| [Amazon Macie](https://aws.amazon.com/macie/) |

## 18. 참고

- [[../s3]]
- [[../02-access/02-block-public-access-and-vpc-endpoint]]
- [[01-encryption-sse-kms-c-bucket-keys]]
- [[02-object-lock-and-mfa-delete]]
