---
title: "troubleshooting · ECR push denied"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, aws, ecr, iam, docker]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# ECR — `denied: User is not authorized`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
denied: User: arn:aws:iam::123:user/ci is not authorized to perform:
  ecr:BatchCheckLayerAvailability on resource: ...
```

또는:
```
no basic auth credentials
unauthorized: authentication required
```

## 원인

1. `aws ecr get-login-password` 안 함 / 만료 (12시간)
2. IAM 권한 부족
3. registry / region 잘못

## 해결

### Docker login (수동)
```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-2

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCT.dkr.ecr.$REGION.amazonaws.com
```

→ 12시간 후 만료. CI 마다 새로.

### IAM policy 부여
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ecr:GetAuthorizationToken",
      "ecr:BatchCheckLayerAvailability",
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage",
      "ecr:InitiateLayerUpload",
      "ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload",
      "ecr:PutImage"
    ],
    "Resource": "arn:aws:ecr:ap-northeast-2:123:repository/myapp"
  }]
}
```

→ AWS managed `AmazonEC2ContainerRegistryPowerUser` 도 OK (전체 repo).

### GitHub Actions (OIDC)
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/github-actions
      aws-region: ap-northeast-2

  - uses: aws-actions/amazon-ecr-login@v2

  - run: |
      docker build -t $ECR_URI:$GITHUB_SHA .
      docker push $ECR_URI:$GITHUB_SHA
```

→ IAM user access key 없이 OIDC 인증.

### Repository policy (cross-account)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::OTHER_ACCT:root" },
    "Action": ["ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage", "ecr:BatchCheckLayerAvailability"]
  }]
}
```

→ 다른 계정에서 pull 허용.

### Public ECR (gallery.ecr.aws)
```bash
aws ecr-public get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin public.ecr.aws
# us-east-1 만 지원
```

## 진단

```bash
# Caller identity
aws sts get-caller-identity

# Repository 확인
aws ecr describe-repositories --region $REGION

# 이전 push 한 image
aws ecr describe-images --repository-name myapp --region $REGION

# Login token 직접
aws ecr get-authorization-token --region $REGION
```

## 재발 방지

- GitHub Actions 는 OIDC 표준 (access key 금지)
- IAM policy 는 least privilege + repo arn 지정
- Lifecycle policy 로 옛 image 자동 정리 (cost / clutter)
- `IMMUTABLE` tag mutability — 같은 tag 재사용 차단

## 관련

- [[devops]] / [[ecs-task-failed-to-start]] / [[docker-arch-mismatch]]
