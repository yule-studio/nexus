---
title: "ECR 실습 2 — Multi-arch + Lifecycle + Cross-region Replication"
kind: practice
status: current
created_at: 2026-05-22T17:56:00+09:00
tags: [aws, ecr, practice, multi-arch, lifecycle, replication]
home_hub: ecr
---

# 실습 2 — Multi-arch + Lifecycle + Replication

> 목표: Multi-arch image + 복잡 Lifecycle + cross-region 자동 replication + EventBridge 알림. 30분.

> 실습 1 의 `lab-app` 위에서 진행.

## 1. Lifecycle Policy 복잡 (multi-rule)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

aws ecr put-lifecycle-policy --repository-name lab-app \
  --lifecycle-policy-text "$(cat <<'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 30 prod- images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod-"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 10 staging- images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["staging-"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "PR images 14d expire",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["pr-*"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 14
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 4,
      "description": "Untagged 7d expire",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
EOF
)"

# Dry-run preview
aws ecr start-lifecycle-policy-preview --repository-name lab-app \
  --lifecycle-policy-text file://lifecycle.json

aws ecr get-lifecycle-policy-preview --repository-name lab-app \
  --query 'previewResults[].imageIds'
```

## 2. Cross-region Replication

```bash
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [{
    "destinations": [
      { "region": "ap-northeast-1", "registryId": "ACCT_ID" }
    ],
    "repositoryFilters": [
      { "filter": "lab-", "filterType": "PREFIX_MATCH" }
    ]
  }]
}' | jq .

# ACCT_ID → 실제 account ID 로 변경
```

## 3. Replication 확인

```bash
# Source push
docker buildx build --platform linux/arm64 \
  -t $ACCT.dkr.ecr.ap-northeast-2.amazonaws.com/lab-app:prod-v3 \
  --push .

sleep 30

# Destination region 확인
aws ecr list-images --repository-name lab-app --region ap-northeast-1
# > prod-v3 이 자동 복제됨
```

## 4. EventBridge — Scan finding 알림

```bash
# SNS topic
aws sns create-topic --name lab-ecr-alerts
SNS_ARN=$(aws sns get-topic-attributes --topic-arn arn:aws:sns:ap-northeast-2:$ACCT:lab-ecr-alerts \
  --query 'Attributes.TopicArn' --output text)
aws sns subscribe --topic-arn $SNS_ARN --protocol email --notification-endpoint you@example.com

# EventBridge rule
aws events put-rule --name lab-ecr-scan-finding \
  --event-pattern '{
    "source": ["aws.ecr"],
    "detail-type": ["ECR Image Scan"],
    "detail": {
      "scan-status": ["COMPLETE"],
      "finding-severity-counts": {
        "HIGH": [{ "numeric": [">=", 1] }]
      }
    }
  }'

aws events put-targets --rule lab-ecr-scan-finding \
  --targets "Id=1,Arn=$SNS_ARN"

# SNS topic policy (EventBridge invoke)
aws sns set-topic-attributes --topic-arn $SNS_ARN --attribute-name Policy \
  --attribute-value "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "events.amazonaws.com"},
    "Action": "sns:Publish",
    "Resource": "$SNS_ARN"
  }]
}
EOF
)"
```

## 5. Scan 결과 트리거

```bash
# 일부러 오래된 image (CVE 있을 가능성)
cat > Dockerfile <<'EOF'
FROM node:14-alpine    # 오래된 Node
RUN apk add --no-cache curl
CMD ["sleep","infinity"]
EOF

docker buildx build --platform linux/arm64 \
  -t $ACCT.dkr.ecr.ap-northeast-2.amazonaws.com/lab-app:prod-vuln \
  --push .

# Scan wait
aws ecr wait image-scan-complete --repository-name lab-app --image-id imageTag=prod-vuln

# Result
aws ecr describe-image-scan-findings --repository-name lab-app --image-id imageTag=prod-vuln \
  --query 'imageScanFindings.findingSeverityCounts'

# 이메일 도착 확인 (HIGH 발견 시)
```

## 6. Cosign 서명 (옵션)

```bash
brew install cosign 2>/dev/null || true

# KMS key (한 번만)
KEY_ID=$(aws kms create-key --description "lab cosign" \
  --key-spec ECC_NIST_P256 --key-usage SIGN_VERIFY \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-cosign --target-key-id $KEY_ID

# Sign
cosign sign --key awskms:///alias/lab-cosign \
  $ACCT.dkr.ecr.ap-northeast-2.amazonaws.com/lab-app:prod-v3 --yes

# Verify
cosign verify --key awskms:///alias/lab-cosign \
  $ACCT.dkr.ecr.ap-northeast-2.amazonaws.com/lab-app:prod-v3

# Repository 안 sig 확인
aws ecr list-images --repository-name lab-app \
  --query 'imageIds[?starts_with(imageTag, `sha256-`)].imageTag'
```

## 7. Pull-through Cache (Docker Hub)

```bash
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix dockerhub \
  --upstream-registry-url registry-1.docker.io

# Pull
docker pull $ACCT.dkr.ecr.ap-northeast-2.amazonaws.com/dockerhub/library/redis:7-alpine

# Cache 자동 생성 확인
aws ecr describe-repositories --query 'repositories[?contains(repositoryName, `dockerhub/`)].repositoryName'
```

## 8. Cost Tag

```bash
aws ecr tag-resource --resource-arn arn:aws:ecr:ap-northeast-2:$ACCT:repository/lab-app \
  --tags Key=Environment,Value=lab Key=Service,Value=demo Key=CostCenter,Value=engineering
```

→ Cost Explorer 에서 service / environment 별 cost visibility.

## 9. Clean up

```bash
# Pull-through cache 삭제
aws ecr delete-pull-through-cache-rule --ecr-repository-prefix dockerhub
aws ecr delete-repository --repository-name dockerhub/library/redis --force

# EventBridge
aws events remove-targets --rule lab-ecr-scan-finding --ids 1
aws events delete-rule --name lab-ecr-scan-finding
aws sns delete-topic --topic-arn $SNS_ARN

# Replication 비활성
aws ecr put-replication-configuration --replication-configuration '{"rules":[]}'

# Repository
aws ecr delete-repository --repository-name lab-app --force
aws ecr delete-repository --repository-name lab-app --region ap-northeast-1 --force

# KMS key 삭제 (스케줄 7일)
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/lab-cosign
```

## 10. 학습 포인트

- Lifecycle 4-rule 의 priority 와 selection
- Dry-run preview
- Cross-region replication (수초)
- EventBridge → SNS 의 자동 알림
- Cosign + AWS KMS 서명
- Pull-through Cache 의 Docker Hub mirror
- Cost Allocation Tag

## 11. 다음

- ▶ [[../ecr]]
- ▶ [[../03-security/02-immutability-and-signing]]
- ▶ [[../../ecs/practice/01-fargate-first-service]]
- ▶ [[../../eks/practice/01-cluster-with-fargate]]

## 12. 참고

- [[../ecr]]
- [[01-create-push-pull]]
- [[../02-operations/02-lifecycle-and-replication]]
- [[../03-security/01-scanning-and-inspector]]
- [[../03-security/02-immutability-and-signing]]
