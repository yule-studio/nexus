---
title: "S3 04-1 — Encryption (SSE-S3 / SSE-KMS / DSSE / Bucket Keys)"
kind: knowledge
status: current
created_at: 2026-05-22T18:56:00+09:00
tags: [aws, s3, security, encryption, kms, sse, bucket-key]
home_hub: s3
---

# S3 Encryption

> 2023+ 기본 SSE-S3 자동 (모든 bucket). SSE-KMS / DSSE-KMS / SSE-C / Client-Side 추가 옵션. Bucket Keys = KMS cost ↓ 99%.

## 1. 4 종류

| | SSE-S3 | SSE-KMS | DSSE-KMS | SSE-C | CSE |
| --- | --- | --- | --- | --- | --- |
| Key 관리 | AWS | AWS+사용자 KMS | AWS+사용자 KMS (dual) | 사용자 (client-provide) | 사용자 (client-side encrypt) |
| Algorithm | AES-256 | AES-256 | AES-256 + 추가 | AES-256 | 사용자 정의 |
| KMS audit | ✗ | ✓ | ✓ | ✗ | ✗ |
| KMS cost | $0 | per request | per request × 2 | $0 | $0 |
| Default (2023+) | ✓ | optional | optional | optional | optional |
| Compliance | low | high | highest (정부) | special | special |
| 권장 | 일반 | sensitive | 정부 / 규제 | special needs | end-to-end |

## 2. SSE-S3 (default)

```bash
# 2023+ 모든 새 bucket 자동
aws s3api get-bucket-encryption --bucket bkurs
# AES256
```

명시:
```bash
aws s3 cp file.json s3://bkurs/ --sse AES256
```

> 비활성 불가 (모든 object 암호화 강제).

## 3. SSE-KMS

```bash
# Bucket default
aws s3api put-bucket-encryption --bucket bkurs \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/bkurs"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

PUT 시:
```bash
aws s3 cp file.json s3://bkurs/ \
  --sse aws:kms \
  --sse-kms-key-id alias/bkurs
```

## 4. KMS key 종류

| | AWS-managed (aws/s3) | Customer-managed (CMK) |
| --- | --- | --- |
| 비용 | 무료 | $1/key/월 |
| Rotation | 자동 1년 | 옵션 |
| Audit | 제한 | full CloudTrail |
| Policy 제어 | ✗ | ✓ |
| Cross-account | 제한 | ✓ |

> production = **CMK 권장** (audit + policy).

## 5. SSE-KMS 의 Permission

Caller (write):
```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject", "kms:GenerateDataKey"],
  "Resource": ["arn:aws:s3:::bkurs/*", "arn:aws:kms:...:key/abc"]
}
```

Caller (read):
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "kms:Decrypt"],
  "Resource": ["arn:aws:s3:::bkurs/*", "arn:aws:kms:...:key/abc"]
}
```

→ KMS key policy 도 caller 허용 필요.

## 6. Bucket Keys (KMS cost ↓)

```json
"BucketKeyEnabled": true
```

```
Without Bucket Key:
  매 object PUT/GET → KMS API 호출 (per-request cost $0.03/10K)

With Bucket Key:
  S3 가 bucket level key 를 short-lived cache (24h)
  → KMS API 호출 99% ↓
  → 비용 99% ↓
```

> **모든 SSE-KMS bucket 에 Bucket Key 활성 권장**.

## 7. DSSE-KMS (Dual-Layer)

```bash
--sse-algorithm aws:kms:dsse
```

→ 2 layer 암호화 (각각 다른 key). 정부 / NSA 권장. 같은 비용 × 2.

## 8. SSE-C (Customer-provided key)

```bash
aws s3 cp file.json s3://bkurs/ \
  --sse-c AES256 \
  --sse-c-key "$(cat key.bin | base64)" \
  --sse-c-key-md5 "$(md5sum key.bin | awk '{print $1}' | xxd -r -p | base64)"
```

- S3 가 key 저장 안 함
- 매 PUT/GET 마다 key 전송
- key 잃으면 데이터 영구 손실
- Replication 안 됨
- 일반 권장 ✗

## 9. CSE — Client-Side Encryption

```python
from boto3 import client
from awscrypto.aws_encryption_sdk import EncryptionSDKClient

sdk = EncryptionSDKClient()
plaintext = open("file.bin","rb").read()
ciphertext, _ = sdk.encrypt(source=plaintext, ...)
client("s3").put_object(Bucket="bkurs", Key="file.bin.enc", Body=ciphertext)
```

→ S3 에 도착 전 client 에서 암호화. End-to-end 보안. 복잡.

## 10. Force encryption — Bucket Policy

```json
{
  "Sid": "DenyUnencryptedPut",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::bkurs/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

→ SSE-KMS 만 허용 (SSE-S3 도 거부).

## 11. Key Policy 예 (CMK)

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
      "Sid": "AllowS3Use",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com",
        "AWS": "arn:aws:iam::ACCT:role/bkurs-app"
      },
      "Action": ["kms:Encrypt","kms:Decrypt","kms:GenerateDataKey*","kms:DescribeKey"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "kms:ViaService": "s3.ap-northeast-2.amazonaws.com" }
      }
    }
  ]
}
```

## 12. Key Rotation

```bash
# Automatic rotation (1년)
aws kms enable-key-rotation --key-id alias/bkurs

# Manual (별 key 로 alias 이동)
aws kms create-key --description "bkurs-2026"
aws kms update-alias --alias-name alias/bkurs --target-key-id new-key-id
```

> Rotation 후 기존 object 도 새 key 로 점진적 re-encrypt 가능 (S3 Batch Operations).

## 13. Cross-Account encryption

Account A bucket + Account B 의 key:
```json
{
  "Effect": "Allow",
  "Action": "kms:Decrypt",
  "Resource": "arn:aws:kms:ap-northeast-2:ACCT_B:key/abc",
  ...
}
```

+ KMS key policy 에 Account A principal 허용.

## 14. CloudTrail audit

```bash
# KMS key 사용 history
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=alias/bkurs \
  --max-items 30 \
  --query 'Events[].{Time:EventTime,Event:EventName,User:Username}'
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Bucket Key 비활성 → KMS cost 폭주 | `BucketKeyEnabled: true` |
| Caller 에 kms:Decrypt 권한 없음 | IAM 추가 |
| Force encryption 정책 없음 | Bucket Policy Deny |
| SSE-C key 분실 | 일반 사용 X |
| Replication + KMS — replica key 권한 누락 | dest KMS key 권한 |
| Cross-account KMS access 한쪽 만 | 양쪽 (key policy + IAM) |

## 16. 다음

- ▶ [[02-object-lock-and-mfa-delete]]
- ▶ [[03-access-analyzer-and-cloudtrail]]
- ▶ [[../../security/kms/kms]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [S3 Encryption 공식](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html) |
| [Bucket Keys](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-key.html) |
| [DSSE-KMS 발표](https://aws.amazon.com/blogs/aws/new-amazon-s3-dual-layer-server-side-encryption-with-keys-stored-in-aws-key-management-service-dsse-kms/) |

## 18. 참고

- [[../s3]]
- [[../02-access/01-bucket-policy-and-acl]]
- [[../../security/kms/kms]]
