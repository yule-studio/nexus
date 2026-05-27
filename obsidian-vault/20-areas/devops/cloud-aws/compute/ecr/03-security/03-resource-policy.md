---
title: "ECR 03-3 — Repository Policy + Registry Policy"
kind: knowledge
status: current
created_at: 2026-05-22T17:52:00+09:00
tags: [aws, ecr, security, iam, repository-policy, registry-policy]
home_hub: ecr
---

# ECR Repository Policy + Registry Policy

> 2 layer 권한 — IAM (caller) + Repository/Registry Policy (resource). Cross-account + Service principal.

## 1. 권한 결합

```
Action 허용 = IAM allow AND Resource Policy allow (cross-account 만)
            = IAM allow OR Resource Policy allow (same account)
```

→ Cross-account 는 **양쪽 모두 통과** 필요.

## 2. Repository Policy

`bkurs-be` repository 만:
```bash
aws ecr set-repository-policy --repository-name bkurs-be \
  --policy-text "$(cat <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgPull",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxx"
        }
      }
    },
    {
      "Sid": "AllowPushFromCI",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/GitHubActionsECRPush"
      },
      "Action": [
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}
EOF
)"
```

## 3. Registry Policy (Account-wide)

```bash
aws ecr put-registry-policy --policy-text '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReplicationAccess",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::987654321098:root" },
    "Action": ["ecr:ReplicateImage","ecr:CreateRepository"],
    "Resource": "*"
  }]
}'
```

> Registry Policy 는 cross-account replication / CreateRepository 만. Repository 단위 권한 = Repository Policy.

## 4. Service principal 권한

ECR 의 ECS / Lambda / SageMaker 직접 접근:
```json
{
  "Sid": "ECSPullAccess",
  "Effect": "Allow",
  "Principal": {
    "Service": "ecs-tasks.amazonaws.com"
  },
  "Action": [
    "ecr:GetDownloadUrlForLayer",
    "ecr:BatchGetImage"
  ],
  "Condition": {
    "StringEquals": {
      "aws:SourceAccount": "123456789012"
    },
    "ArnLike": {
      "aws:SourceArn": "arn:aws:ecs:ap-northeast-2:123456789012:cluster/*"
    }
  }
}
```

> 거의 사용 안 함 — ECS Execution Role 의 IAM 으로 처리. Service principal 은 backup.

## 5. AWS managed policy (consumer)

| Policy | 권한 |
| --- | --- |
| `AmazonEC2ContainerRegistryFullAccess` | 모든 ECR action |
| `AmazonEC2ContainerRegistryPowerUser` | push + pull (admin 제외) |
| `AmazonEC2ContainerRegistryReadOnly` | pull only |
| `AmazonEC2ContainerRegistryPullOnly` | pull only (2023+) |

권장 — production task 의 Execution Role 에 `AmazonEC2ContainerRegistryReadOnly`.

## 6. Tag-based ABAC

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:BatchGetImage",
    "ecr:GetDownloadUrlForLayer"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Environment": "${aws:PrincipalTag/Environment}"
    }
  }
}
```

→ ECR Repository 에 `Environment=prod` tag + Caller IAM Role 에 `Environment=prod` tag 일 때만.

## 7. Endpoint Policy (VPC PrivateLink)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPullOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

VPC Endpoint 에 attach → VPC 안에서는 push 안 됨 (보안).

## 8. Permission Boundary

ECR push 권한을 제한:
```json
{
  "Effect": "Deny",
  "Action": "ecr:DeleteRepository",
  "Resource": "arn:aws:ecr:*:*:repository/prod-*"
}
```

## 9. CloudTrail audit

```bash
# 누가 image push
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutImage \
  --max-items 30

# 누가 lifecycle 변경
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutLifecyclePolicy

# 누가 repository 삭제
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteRepository
```

## 10. KMS — image at-rest 암호화

```bash
aws ecr create-repository --repository-name bkurs-be \
  --encryption-configuration "encryptionType=KMS,kmsKey=alias/ecr"
```

Repository 생성 후 변경 불가.

Consumer (pull side) Execution Role 의 `kms:Decrypt`:
```json
{
  "Effect": "Allow",
  "Action": "kms:Decrypt",
  "Resource": "arn:aws:kms:...:key/abc",
  "Condition": {
    "StringEquals": { "kms:ViaService": "ecr.ap-northeast-2.amazonaws.com" }
  }
}
```

## 11. Cross-Org / Cross-Account 권한 정리

| 시나리오 | Repository Policy | Consumer IAM |
| --- | --- | --- |
| 같은 account push | not needed (IAM 만) | ✓ |
| 같은 account pull | not needed (IAM 만) | ✓ |
| Cross-account pull | ✓ | ✓ |
| Cross-account replication | Registry Policy ✓ | n/a |
| Cross-org public | Repository Policy with PrincipalOrgID condition | ✓ |
| ECR Public anonymous | n/a | n/a |

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Repository Policy Principal `*` (open to world) | Condition (Account / Org) |
| Cross-account IAM 만 (Resource Policy 누락) | 양쪽 |
| Service principal SourceArn 누락 | confused deputy 차단 |
| KMS encryption 변경 시도 (불가) | repository 처음에 결정 |
| Registry Policy 와 Repository Policy 혼동 | 용도 분리 |

## 13. 다음

- ▶ [[../practice/01-create-push-pull]]
- ▶ [[../practice/02-multi-arch-and-lifecycle]]
- ▶ [[../../iam/iam]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [Repository Policy](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-policies.html) |
| [Registry Policy](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry-permissions.html) |

## 15. 참고

- [[../ecr]]
- [[../02-operations/03-cross-account-and-region]]
- [[../../iam/iam]]
