---
title: "S3 02-1 — Bucket Policy + IAM + ACL (legacy)"
kind: knowledge
status: current
created_at: 2026-05-22T18:40:00+09:00
tags: [aws, s3, access, bucket-policy, iam, acl]
home_hub: s3
---

# S3 Bucket Policy + IAM + ACL

> 3 layer — IAM (caller) + Bucket Policy (resource) + ACL (legacy). ACL 권장 ✗, Bucket Policy + IAM 사용.

## 1. 평가 흐름

```
Request → 
  1. Block Public Access (account/bucket level) — 가장 강함
  2. IAM Identity (caller) — Allow/Deny
  3. Bucket Policy — Allow/Deny
  4. Object ACL (있을 시) — legacy

→ 어느 하나라도 Deny = 거부
→ Block Public Access 가 public access 차단
```

## 2. IAM Policy (caller side)

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": [
    "arn:aws:s3:::bkurs-uploads/*"
  ]
}
```

| Action | 의미 |
| --- | --- |
| `s3:ListBucket` | bucket 안 object list (Resource = bucket ARN) |
| `s3:GetObject` | download |
| `s3:PutObject` | upload |
| `s3:DeleteObject` | delete |
| `s3:GetObjectVersion` | 특정 version download |
| `s3:PutObjectTagging` | tag |
| `s3:PutObjectAcl` | ACL (지양) |
| `s3:GetBucketLocation` | region 조회 |

## 3. Bucket Policy (resource side)

```bash
aws s3api put-bucket-policy --bucket bkurs-uploads --policy "$(cat <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::bkurs-uploads",
        "arn:aws:s3:::bkurs-uploads/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    },
    {
      "Sid": "AllowCloudFrontPull",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::bkurs-uploads/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::ACCT:distribution/DIST_ID"
        }
      }
    }
  ]
}
EOF
)"
```

### Bucket Policy 의 사용처
- Service 의 access 허용 (CloudFront / Replication / Logging)
- Cross-account access
- HTTP/HTTPS 강제
- IP 제한 (회사 network 만)
- VPC Endpoint 만 허용

## 4. 흔한 Bucket Policy 패턴

### HTTPS 강제
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::bkurs/*", "arn:aws:s3:::bkurs"],
  "Condition": { "Bool": { "aws:SecureTransport": "false" } }
}
```

### IP 제한
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::bkurs/*",
  "Condition": {
    "NotIpAddress": { "aws:SourceIp": ["10.0.0.0/8","203.0.113.0/24"] }
  }
}
```

### VPC Endpoint 만
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::bkurs/*",
  "Condition": {
    "StringNotEquals": { "aws:SourceVpce": "vpce-xxxxx" }
  }
}
```

### Encryption 강제
```json
{
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

### Cross-Account Read
```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::OTHER_ACCT:root" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::bkurs/public/*"
}
```

## 5. Bucket Policy size limit

```
Bucket policy:  20 KB
Statement 수:   따로 한도 없음 (20 KB 안에서)
```

> 큰 policy = Identity-based IAM 으로.

## 6. ACL (legacy, 사용 안 함)

```bash
# Object 의 ACL
aws s3api put-object-acl --bucket bkurs --key file.json --acl public-read
```

Canned ACL:
- `private` (default)
- `public-read`
- `public-read-write`
- `authenticated-read`
- `bucket-owner-read`
- `bucket-owner-full-control`
- `aws-exec-read`
- `log-delivery-write` (S3 access log only)

> **2023+ AWS 권장 — ACL disable + Bucket Policy 만**.

```bash
aws s3api put-bucket-ownership-controls --bucket bkurs \
  --ownership-controls 'Rules=[{ObjectOwnership=BucketOwnerEnforced}]'
```

→ ACL disabled, 모든 object 가 bucket owner.

## 7. Object Ownership 3 mode

| Mode | 의미 |
| --- | --- |
| `ObjectWriter` | object writer 가 owner (legacy default) |
| `BucketOwnerPreferred` | bucket owner 가 owner (uploader 가 bucket-owner-full-control 명시) |
| `BucketOwnerEnforced` ★ | bucket owner 가 owner (ACL disabled, 2023 권장) |

## 8. ABAC — Tag 기반

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::bkurs/*",
  "Condition": {
    "StringEquals": {
      "s3:ExistingObjectTag/Environment": "${aws:PrincipalTag/Environment}"
    }
  }
}
```

→ Caller IAM tag 가 object tag 와 같을 때만.

## 9. Cross-account 표준 패턴

Account A 의 bucket → Account B 의 caller:

### A 의 Bucket Policy
```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::ACCT_B:role/Caller" },
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::bkurs",
    "arn:aws:s3:::bkurs/*"
  ]
}
```

### B 의 IAM Role
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::bkurs",
    "arn:aws:s3:::bkurs/*"
  ]
}
```

→ **양쪽 모두 통과** 필요.

## 10. Permission Boundary / SCP

```json
{
  "Effect": "Deny",
  "Action": "s3:DeleteBucket",
  "Resource": "arn:aws:s3:::bkurs-prod-*"
}
```

→ admin 도 prod- bucket 삭제 못 함 (defense in depth).

## 11. Confused Deputy (Service Principal)

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::bkurs/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::ACCT:distribution/DIST_ID"
    }
  }
}
```

→ SourceArn 명시 안 하면 다른 사용자의 CloudFront 가 접근 가능.

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Bucket Policy Principal `*` (public) | Block Public Access |
| ACL 사용 | BucketOwnerEnforced + Bucket Policy |
| SourceArn 누락 (confused deputy) | 명시 |
| Cross-account 한쪽 만 | 양쪽 |
| HTTPS 강제 안 함 | SecureTransport condition |
| Bucket Policy 너무 큼 | IAM identity policy |

## 13. 다음

- ▶ [[02-block-public-access-and-vpc-endpoint]]
- ▶ [[03-presigned-url-and-cors]]
- ▶ [[../04-security/01-encryption-sse-kms-c-bucket-keys]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [S3 Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-overview.html) |
| [Bucket Policy 공식](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html) |
| [Object Ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html) |

## 15. 참고

- [[../s3]]
- [[../../iam/iam]]
