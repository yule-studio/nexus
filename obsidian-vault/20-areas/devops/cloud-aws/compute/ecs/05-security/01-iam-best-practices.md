---
title: "ECS 05-1 — IAM Best Practices"
kind: knowledge
status: current
created_at: 2026-05-22T14:10:00+09:00
tags: [aws, ecs, security, iam]
home_hub: ecs
---

# ECS IAM Best Practices

> 3 role 분리 + least-privilege + boundary. [[../02-task-definition/06-iam-roles]] 의 보안 심화.

## 1. 3 role 위치 재확인

```
Execution Role  → ECS Agent (pull / log / secret 가져오기)
Task Role       → application (DB / S3 / API 호출)
Instance Role   → EC2 (cluster register, EC2 launch 만)
```

## 2. Execution Role — minimal

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [ "logs:CreateLogStream", "logs:PutLogEvents", "logs:CreateLogGroup" ],
      "Resource": "arn:aws:logs:ap-northeast-2:123:log-group:/ecs/bkurs-be*"
    },
    {
      "Effect": "Allow",
      "Action": "ssm:GetParameters",
      "Resource": "arn:aws:ssm:ap-northeast-2:123:parameter/bkurs-be/prod/*"
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:ap-northeast-2:123:key/abc-...",
      "Condition": { "StringEquals": { "kms:ViaService": "ssm.ap-northeast-2.amazonaws.com" } }
    }
  ]
}
```

> AWS managed `AmazonECSTaskExecutionRolePolicy` 는 시작용. **production 은 inline + 리소스 ARN 한정**.

## 3. Task Role — least-privilege

bkurs-be 의 예 (S3 + SES 만):
```json
[
  {
    "Effect": "Allow",
    "Action": ["s3:GetObject","s3:PutObject"],
    "Resource": "arn:aws:s3:::bkurs-backup-prod/*"
  },
  {
    "Effect": "Allow",
    "Action": "ses:SendEmail",
    "Resource": "*",
    "Condition": {
      "StringEquals": { "ses:FromAddress": "noreply@bkurs.com" }
    }
  }
]
```

> wildcard `s3:*` / `Resource: "*"` 절대 금지.

## 4. Permission Boundary

권한 escalation 방지:
```json
{
  "Effect": "Deny",
  "Action": ["iam:CreateRole", "iam:AttachRolePolicy"],
  "Resource": "*"
}
```

Boundary attach:
```bash
aws iam put-role-permissions-boundary \
  --role-name BkursTaskRole \
  --permissions-boundary arn:aws:iam::123:policy/BkursBoundary
```

## 5. Trust Policy — confused deputy 방지

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "ArnLike": { "aws:SourceArn": "arn:aws:ecs:ap-northeast-2:123:*" },
      "StringEquals": { "aws:SourceAccount": "123456789012" }
    }
  }]
}
```

## 6. SCP (Service Control Policy)

org level — 모든 task role 에 강제:
```json
{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {
    "Bool": { "aws:SecureTransport": "false" }
  }
}
```

## 7. Cross-Account ECR pull

Account A image → Account B ECS:
- ECR repository policy: Allow Account B principal
- Execution Role: ecr:Get* on cross-account repo ARN

## 8. IAM Access Analyzer

Account 의 모든 IAM 정책 자동 분석 — wildcard / public 노출 탐지.

## 9. ECS Task IAM Role 검증

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123:role/BkursTaskRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::bkurs-backup-prod/foo.txt
```

## 10. Pod Identity (EKS) vs Task Role (ECS)

| | ECS Task Role | EKS Pod Identity |
| --- | --- | --- |
| 구조 | IMDS v2 → STS | IRSA 또는 Pod Identity Agent |
| audit | CloudTrail | CloudTrail |

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `AdministratorAccess` to Task Role | least-privilege |
| Execution / Task Role 혼용 | 분리 |
| AWS managed policy 채로 production | inline + ARN 한정 |
| boundary 없음 | escalation 차단 |
| confused deputy | SourceArn / SourceAccount Condition |

## 12. 다음

- ▶ [[02-network-isolation]]
- ▶ [[03-secret-management]]
- ▶ [[04-image-security]]

## 13. 영상 / 자료

| 자료 |
| --- |
| [IAM Best Practices — AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) |
| [Access Analyzer](https://aws.amazon.com/iam/access-analyzer/) |

## 14. 참고

- [[../ecs]]
- [[../02-task-definition/06-iam-roles]]
- [[../../security-and-iam/iam/iam]]
