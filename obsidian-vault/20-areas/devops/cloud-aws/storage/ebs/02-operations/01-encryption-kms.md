---
title: "EBS 02-1 — Encryption + KMS"
kind: knowledge
status: current
created_at: 2026-05-22T19:21:00+09:00
tags: [aws, ebs, operations, encryption, kms, cmk]
home_hub: ebs
---

# EBS Encryption + KMS

> AES-256 + KMS (CMK 또는 aws/ebs). Default Encryption + Cross-account / Cross-region snapshot.

## 1. Default Encryption (account-wide)

```bash
aws ec2 enable-ebs-encryption-by-default

# 검증
aws ec2 get-ebs-encryption-by-default
```

→ 새 volume / snapshot 자동 암호화. **권장 — 모든 account 활성**.

## 2. Default KMS Key

```bash
# AWS managed (aws/ebs)
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/aws/ebs

# CMK (customer-managed)
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/bkurs-ebs
```

| | AWS managed (aws/ebs) | CMK |
| --- | --- | --- |
| 비용 | 무료 | $1/key/월 |
| Rotation | 1년 자동 | 옵션 |
| Audit | 제한 | full CloudTrail |
| Policy 제어 | ✗ | ✓ |
| Cross-account | 제한 | ✓ |

> production = **CMK 권장**.

## 3. Volume 단위 encryption

```bash
# 명시
aws ec2 create-volume --availability-zone ap-northeast-2a \
  --size 100 --volume-type gp3 \
  --encrypted \
  --kms-key-id alias/bkurs-ebs
```

```typescript
new ec2.Volume(this, 'Data', {
  ...
  encrypted: true,
  encryptionKey: ec2.Key.fromAlias(this, 'Key', 'alias/bkurs-ebs'),
});
```

## 4. Encryption 의 범위

- 영구 volume data
- All snapshot (자동)
- Snapshot copy
- AMI built from encrypted volume
- Volume-attached EC2 의 모든 traffic (EBS → EC2)
- Boot volume (Nitro 만)

## 5. Re-encryption (key 변경)

같은 volume 의 key 직접 변경 ✗. 절차:
```bash
# 1. Snapshot
aws ec2 create-snapshot --volume-id vol-xxx

# 2. Snapshot copy with new key
aws ec2 copy-snapshot --source-region ap-northeast-2 \
  --source-snapshot-id snap-xxx \
  --encrypted --kms-key-id alias/bkurs-new

# 3. New volume from copied snapshot
aws ec2 create-volume --snapshot-id snap-new --availability-zone ... \
  --kms-key-id alias/bkurs-new

# 4. Old volume detach + new attach + delete old
```

## 6. KMS Key Policy 권장

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableAdmin",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCT:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowEBSUse",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey*",
        "kms:ReEncrypt*",
        "kms:CreateGrant",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "kms:ViaService": "ec2.ap-northeast-2.amazonaws.com" },
        "StringEquals": { "kms:CallerAccount": "ACCT" }
      }
    }
  ]
}
```

## 7. Cross-account Snapshot 공유

```bash
# Source account
aws ec2 modify-snapshot-attribute --snapshot-id snap-xxx \
  --create-volume-permission "Add=[{UserId=OTHER_ACCT}]"

# KMS key 도 cross-account grant
aws kms create-grant --key-id alias/bkurs-ebs \
  --grantee-principal arn:aws:iam::OTHER_ACCT:role/ImportRole \
  --operations Decrypt CreateGrant DescribeKey
```

Target account 의 IAM 권한:
```json
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt","kms:CreateGrant"],
  "Resource": "arn:aws:kms:ap-northeast-2:SOURCE_ACCT:key/abc"
}
```

> **양쪽 모두 grant 필요**.

## 8. Cross-region Snapshot copy

```bash
aws ec2 copy-snapshot --source-region ap-northeast-2 \
  --source-snapshot-id snap-xxx \
  --destination-region ap-northeast-1 \
  --encrypted --kms-key-id arn:aws:kms:ap-northeast-1:ACCT:key/abc
```

→ destination region 의 key 로 re-encrypt. source key 와 다른 region.

## 9. AMI from Encrypted Volume

- 자동 암호화 snapshot 생성
- AMI 사용 시 같은 KMS key 권한 필요
- 공유 시 KMS grant 추가

## 10. EBS Encryption + DLM

```bash
aws dlm create-lifecycle-policy --description "encrypted daily" \
  --execution-role-arn arn:iam::...:role/dlm-role \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "TargetTags": [{ "Key": "Backup", "Value": "daily" }],
    "Parameters": { "ExcludeBootVolume": false },
    "Schedules": [{
      "Name": "daily",
      "CreateRule": { "Interval": 24, "IntervalUnit": "HOURS" },
      "RetainRule": { "Count": 14 },
      "CrossRegionCopyRules": [{
        "Target": "ap-northeast-1",
        "Encrypted": true,
        "CmkArn": "arn:aws:kms:ap-northeast-1:ACCT:key/abc",
        "RetainRule": { "Interval": 30, "IntervalUnit": "DAYS" }
      }]
    }]
  }'
```

→ daily 자동 + DR cross-region.

## 11. Encryption 의 성능 영향

```
Nitro instance:
  Hardware encryption acceleration (AES-NI)
  → < 1% overhead
  
Xen-style legacy:
  ~10-15% overhead (드물게 사용)
```

→ **거의 무료**. 항상 활성.

## 12. Compliance — FIPS 140-2

```
KMS endpoint:
- 일반: kms.<region>.amazonaws.com
- FIPS: kms-fips.<region>.amazonaws.com (정부)
```

```python
import boto3
kms = boto3.client("kms", endpoint_url="https://kms-fips.us-east-1.amazonaws.com")
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Default Encryption 비활성 | enable account-wide |
| AWS managed key (aws/ebs) production | CMK |
| KMS key 권한 누락 → attach 실패 | `kms:Decrypt` + `kms:CreateGrant` |
| Cross-account snapshot 공유 시 grant 누락 | 둘 다 |
| AMI 공유 시 KMS grant 누락 | grant 추가 |
| Key rotation 후 cache 갱신 안 됨 | alias 사용 |

## 14. CloudTrail audit

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=alias/bkurs-ebs \
  --max-items 30 \
  --query 'Events[].{Time:EventTime,Event:EventName,User:Username}'
```

## 15. 다음

- ▶ [[02-multi-attach-and-fast-snapshot-restore]]
- ▶ [[03-performance-and-monitoring]]
- ▶ [[../../security/kms/kms]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [EBS Encryption](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html) |
| [KMS for EBS](https://docs.aws.amazon.com/kms/latest/developerguide/services-ebs.html) |

## 17. 참고

- [[../ebs]]
- [[../01-fundamentals/02-snapshot-and-ami]]
- [[../../security/kms/kms]]
