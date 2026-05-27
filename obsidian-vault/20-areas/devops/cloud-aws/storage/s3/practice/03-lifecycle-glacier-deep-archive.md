---
title: "S3 실습 3 — Lifecycle + Glacier Deep Archive + Restore + Replication"
kind: practice
status: current
created_at: 2026-05-22T19:06:00+09:00
tags: [aws, s3, practice, lifecycle, glacier, replication, batch]
home_hub: s3
---

# 실습 3 — Lifecycle + Glacier Deep Archive + Replication

> 목표: 복잡 Lifecycle (Standard → IA → Glacier → Deep) + Replication + Glacier Restore + Object Lock. 60분.

## 1. Bucket (versioning + Object Lock)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
SRC=lab-archive-src-$ACCT-$(date +%s)
DST=lab-archive-dst-$ACCT-$(date +%s)

aws s3api create-bucket --bucket $SRC \
  --object-lock-enabled-for-bucket \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2

aws s3api create-bucket --bucket $DST \
  --object-lock-enabled-for-bucket \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1

# Versioning
for B in $SRC $DST; do
  REG=$([ "$B" = "$DST" ] && echo "ap-northeast-1" || echo "ap-northeast-2")
  aws s3api put-bucket-versioning --bucket $B \
    --versioning-configuration Status=Enabled --region $REG
done

# Encryption (default 2023+)
aws s3api put-bucket-encryption --bucket $SRC \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" },
      "BucketKeyEnabled": true
    }]
  }'
```

## 2. Complex Lifecycle Policy

```bash
aws s3api put-bucket-lifecycle-configuration --bucket $SRC \
  --lifecycle-configuration "$(cat <<'EOF'
{
  "Rules": [
    {
      "ID": "main-flow",
      "Status": "Enabled",
      "Filter": { "Prefix": "data/" },
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 2555 }
    },
    {
      "ID": "noncurrent-cleanup",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionTransitions": [
        { "NoncurrentDays": 30, "StorageClass": "GLACIER" }
      ],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 }
    },
    {
      "ID": "abort-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    },
    {
      "ID": "expire-delete-markers",
      "Status": "Enabled",
      "Filter": {},
      "ExpiredObjectDeleteMarker": true
    },
    {
      "ID": "logs-fast-archive",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 7, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 365 }
    }
  ]
}
EOF
)"
```

## 3. Test object upload

```bash
# Standard
echo "report v1" > report.txt
aws s3 cp report.txt s3://$SRC/data/report.txt

# Direct Glacier Deep Archive
echo "archive log" > archive.log
aws s3 cp archive.log s3://$SRC/logs/archive.log \
  --storage-class DEEP_ARCHIVE

# Versioning (v2)
echo "report v2" > report.txt
aws s3 cp report.txt s3://$SRC/data/report.txt
```

## 4. Replication 활성

```bash
# Replication Role
aws iam create-role --role-name lab-s3-replication-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"s3.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name lab-s3-replication-role --policy-name p --policy-document "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [
    { \"Effect\": \"Allow\", \"Action\": [\"s3:GetReplicationConfiguration\",\"s3:ListBucket\"], \"Resource\": \"arn:aws:s3:::$SRC\" },
    { \"Effect\": \"Allow\", \"Action\": [\"s3:GetObjectVersionForReplication\",\"s3:GetObjectVersionAcl\",\"s3:GetObjectVersionTagging\"], \"Resource\": \"arn:aws:s3:::$SRC/*\" },
    { \"Effect\": \"Allow\", \"Action\": [\"s3:ReplicateObject\",\"s3:ReplicateDelete\",\"s3:ReplicateTags\"], \"Resource\": \"arn:aws:s3:::$DST/*\" }
  ]
}"

aws s3api put-bucket-replication --bucket $SRC --replication-configuration "{
  \"Role\": \"arn:aws:iam::$ACCT:role/lab-s3-replication-role\",
  \"Rules\": [{
    \"ID\": \"to-tokyo\",
    \"Status\": \"Enabled\",
    \"Priority\": 1,
    \"DeleteMarkerReplication\": { \"Status\": \"Enabled\" },
    \"Filter\": {},
    \"Destination\": {
      \"Bucket\": \"arn:aws:s3:::$DST\",
      \"StorageClass\": \"GLACIER\",
      \"ReplicationTime\": { \"Status\": \"Enabled\", \"Time\": { \"Minutes\": 15 } },
      \"Metrics\": { \"Status\": \"Enabled\", \"EventThreshold\": { \"Minutes\": 15 } }
    }
  }]
}"
```

## 5. Object Lock 적용

```bash
# Source 의 새 object 에 Compliance retention 7d (테스트)
RETENTION_DATE=$(date -u -v +7d +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null || date -u -d "+7 days" +"%Y-%m-%dT%H:%M:%SZ")

aws s3api put-object --bucket $SRC --key data/locked.txt \
  --body report.txt \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date "$RETENTION_DATE"

# 삭제 시도 (실패)
aws s3api delete-object --bucket $SRC --key data/locked.txt \
  --version-id $(aws s3api list-object-versions --bucket $SRC --prefix data/locked.txt --query 'Versions[0].VersionId' --output text) \
  2>&1 | head -5
# AccessDenied
```

## 6. Glacier Restore 시뮬

```bash
# Direct deep archive object 의 restore
aws s3api restore-object --bucket $SRC --key logs/archive.log \
  --restore-request '{
    "Days": 7,
    "GlacierJobParameters": { "Tier": "Bulk" }
  }'

# 진행 확인
aws s3api head-object --bucket $SRC --key logs/archive.log \
  --query 'Restore'
# 'ongoing-request="true"'  → restore in progress (~12-48h for Deep)

# 완료 후 (12h+)
aws s3 cp s3://$SRC/logs/archive.log .
```

> Deep Archive bulk = 12-48 시간. 실습 short — Standard tier 도 시도:
```bash
# Glacier (not Deep) 의 standard restore (3-5h)
aws s3 cp test.bin s3://$SRC/data/test.bin --storage-class GLACIER
sleep 5
aws s3api restore-object --bucket $SRC --key data/test.bin \
  --restore-request '{
    "Days": 7,
    "GlacierJobParameters": { "Tier": "Expedited" }
  }'
```

## 7. Inventory daily

```bash
INV=lab-inventory-$ACCT-$(date +%s)
aws s3 mb s3://$INV

aws s3api put-bucket-policy --bucket $INV --policy "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [{
    \"Effect\": \"Allow\",
    \"Principal\": { \"Service\": \"s3.amazonaws.com\" },
    \"Action\": \"s3:PutObject\",
    \"Resource\": \"arn:aws:s3:::$INV/*\",
    \"Condition\": {
      \"ArnLike\": { \"aws:SourceArn\": \"arn:aws:s3:::$SRC\" },
      \"StringEquals\": { \"aws:SourceAccount\": \"$ACCT\" }
    }
  }]
}"

aws s3api put-bucket-inventory-configuration --bucket $SRC \
  --id daily-inventory \
  --inventory-configuration "{
    \"Id\": \"daily-inventory\",
    \"IsEnabled\": true,
    \"Destination\": { \"S3BucketDestination\": { \"Bucket\": \"arn:aws:s3:::$INV\", \"Format\": \"CSV\" } },
    \"Schedule\": { \"Frequency\": \"Daily\" },
    \"IncludedObjectVersions\": \"All\",
    \"OptionalFields\": [\"Size\",\"StorageClass\",\"LastModifiedDate\",\"ETag\",\"EncryptionStatus\",\"ObjectLockRetainUntilDate\"]
  }"
```

## 8. CloudWatch Replication metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=$SRC \
              Name=DestinationBucket,Value=$DST \
              Name=RuleId,Value=to-tokyo \
  --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time   $(date -u                 +"%Y-%m-%dT%H:%M:%SZ") \
  --period 300 --statistics Average,Maximum
```

## 9. Storage Class 변화 (Lifecycle 결과 시뮬)

```bash
# 강제 Standard-IA copy (lifecycle 30d 시뮬)
aws s3 cp s3://$SRC/data/report.txt s3://$SRC/data/report.txt \
  --storage-class STANDARD_IA --metadata-directive REPLACE

aws s3api head-object --bucket $SRC --key data/report.txt \
  --query 'StorageClass'
```

## 10. Clean up

```bash
# Object Lock retention 만료 대기 또는 강제로 정지
aws s3api put-bucket-replication --bucket $SRC --replication-configuration '{"Role":"","Rules":[]}' 2>/dev/null || true
aws s3api delete-bucket-replication --bucket $SRC 2>/dev/null || true

# All version + delete marker
aws s3api delete-objects --bucket $SRC --delete "$(aws s3api list-object-versions --bucket $SRC \
  --query 'Versions[?ObjectLockMode!=`COMPLIANCE`].{Key:Key,VersionId:VersionId}' --output json | jq '{Objects:.}')"  2>/dev/null

# Lock 안 풀린 object 는 만료 후 다시
aws s3 rb s3://$SRC --force 2>/dev/null || echo "wait for retention expiry"

aws s3 rb s3://$DST --force --region ap-northeast-1
aws s3 rb s3://$INV --force

aws iam delete-role-policy --role-name lab-s3-replication-role --policy-name p
aws iam delete-role --role-name lab-s3-replication-role
```

## 11. 학습 포인트

- Lifecycle 의 4 rule (transition / noncurrent / multipart / delete marker)
- Glacier 3 tier 의 비용 / 속도
- Restore 의 lifecycle (7d 동안 Standard)
- Object Lock Compliance 의 영구 보존
- Replication + RTC (15분 SLA)
- Cross-region Storage Class 변환 (Replication 의 옵션)
- Inventory 의 daily 활용 (Athena query 가능)
- Replication metric (CloudWatch)

## 12. 다음

- ▶ [[../s3]]
- ▶ [[../../ebs/ebs]]
- ▶ [[../../efs/efs]]

## 13. 참고

- [[../s3]]
- [[../03-lifecycle/01-lifecycle-policy-and-tiering]]
- [[../03-lifecycle/02-replication-and-batch-ops]]
- [[../04-security/02-object-lock-and-mfa-delete]]
