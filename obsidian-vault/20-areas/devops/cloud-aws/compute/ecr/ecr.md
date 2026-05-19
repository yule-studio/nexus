---
title: "ECR — Elastic Container Registry"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:02:00+09:00
tags: [devops, cloud-aws, ecr, container]
---

# ECR — Elastic Container Registry

**[[cloud-aws|↑ cloud-aws]]**

---

## 1. 무엇

```
AWS managed Docker / OCI image registry:
  - private repository
  - IAM 통합 (push/pull 권한)
  - encryption at rest (KMS)
  - vulnerability scan (Trivy / native)
  - replication (cross-region)
  - lifecycle policy (옛 image 자동 삭제)

비용:
  - storage: $0.10/GB-mo
  - data transfer (cross-region): $0.02/GB
  - public repository = free
```

→ Docker Hub 대안. AWS 환경의 표준.

---

## 2. private vs public

```
Private ECR:
  - account / IAM 권한 필요
  - VPC 안 endpoint 가능
  - 대부분 회사가 사용

Public ECR:
  - 무료
  - 모두 pull 가능
  - OSS 배포 용
  - Docker Hub rate limit 회피 (★)
```

---

## 3. repository 생성

```bash
# 생성
aws ecr create-repository \
    --repository-name my-app \
    --region ap-northeast-2 \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=KMS

# 또는 IaC (Terraform)
resource "aws_ecr_repository" "my_app" {
  name                 = "my-app"
  image_tag_mutability = "IMMUTABLE"      # ★ tag 덮어쓰기 금지

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }
}
```

→ `IMMUTABLE` tag = 같은 tag 덮어쓰기 X (★ supply chain 보안).

---

## 4. push / pull

```bash
# 1. login (ECR 의 temporary password)
aws ecr get-login-password --region ap-northeast-2 \
    | docker login --username AWS \
                   --password-stdin 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com

# 2. tag
docker tag my-app:latest \
    123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:1.0

# 3. push
docker push \
    123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:1.0

# 4. pull
docker pull 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:1.0
```

→ token 의 TTL 12 hour.

---

## 5. multi-arch (★ ARM + AMD64)

```bash
# buildx 가 manifest 자동
docker buildx create --use --name multi

docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:1.0 \
    --push \
    .

# 사용자의 platform 에 맞는 image 자동 fetch
```

---

## 6. IAM 권한

```json
// ECR push (CI 의 IAM Role)
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage",
                "ecr:BatchGetImage",
                "ecr:GetDownloadUrlForLayer"
            ],
            "Resource": "arn:aws:ecr:ap-northeast-2:123:repository/my-app"
        }
    ]
}
```

```json
// ECR pull only (EKS node / Lambda)
{
    "Effect": "Allow",
    "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
    ],
    "Resource": "*"
}
```

---

## 7. repository policy (cross-account)

```json
// account A 의 ECR 을 account B 가 pull
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowOtherAccount",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::222222222222:root"
            },
            "Action": [
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer"
            ]
        }
    ]
}
```

→ central account 의 ECR → 모든 account 가 pull.

---

## 8. lifecycle policy (★ 중요)

```json
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "최근 10 개의 image 유지 (semver tag)",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["v"],
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": {"type": "expire"}
        },
        {
            "rulePriority": 2,
            "description": "untagged image 7일 후 삭제",
            "selection": {
                "tagStatus": "untagged",
                "countType": "sinceImagePushed",
                "countUnit": "days",
                "countNumber": 7
            },
            "action": {"type": "expire"}
        }
    ]
}
```

```bash
aws ecr put-lifecycle-policy \
    --repository-name my-app \
    --lifecycle-policy-text file://lifecycle.json
```

→ storage cost 폭주 방지. 매일 push 면 빠르게 GB+.

---

## 9. image scan (vulnerability)

```bash
# Basic scan (CVE 데이터베이스 기반, 무료)
aws ecr describe-image-scan-findings \
    --repository-name my-app \
    --image-id imageTag=1.0

# Enhanced scan (Amazon Inspector, 더 정확, 유료)
aws ecr put-registry-scanning-configuration \
    --scan-type ENHANCED \
    --rules '[{
        "scanFrequency": "CONTINUOUS_SCAN",
        "repositoryFilters": [{"filter": "*", "filterType": "WILDCARD"}]
    }]'
```

→ Basic = OS 패키지만. Enhanced = OS + application (Java / Python / Go ...).

---

## 10. replication (★ DR / multi-region)

```hcl
resource "aws_ecr_replication_configuration" "main" {
  replication_configuration {
    rule {
      destination {
        region      = "us-east-1"
        registry_id = "123456789012"
      }
      destination {
        region      = "us-west-2"
        registry_id = "123456789012"
      }
      repository_filter {
        filter      = "prod-*"
        filter_type = "PREFIX_MATCH"
      }
    }
  }
}
```

→ push 시 자동 다른 region 으로 복제. DR + cross-region pull 빠름.

---

## 11. pull-through cache (★ 2022+)

```
ECR 이 Docker Hub / Quay / public ECR 의 cache 역할:

설정:
  aws ecr create-pull-through-cache-rule \
      --ecr-repository-prefix dockerhub \
      --upstream-registry-url public.ecr.aws \
      --credential-arn arn:aws:secretsmanager:...

사용:
  docker pull 123.dkr.ecr.ap-northeast-2.amazonaws.com/dockerhub/library/nginx:1.25

  → 첫 pull = upstream 에서 가져옴 + ECR 저장
  → 이후 = ECR 에서 (Docker Hub rate limit 회피 ★)
```

→ Docker Hub 의 rate limit 회피 + private network egress 절감.

---

## 12. VPC Endpoint (★ private)

```bash
# ECR API endpoint
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxx \
    --service-name com.amazonaws.ap-northeast-2.ecr.api \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-xxx subnet-yyy \
    --security-group-ids sg-xxx

# ECR DKR endpoint (image layer)
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxx \
    --service-name com.amazonaws.ap-northeast-2.ecr.dkr \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-xxx subnet-yyy

# S3 Gateway endpoint (ECR 의 layer 는 S3 에)
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxx \
    --service-name com.amazonaws.ap-northeast-2.s3 \
    --vpc-endpoint-type Gateway \
    --route-table-ids rtb-xxx
```

→ NAT Gateway 없이 ECR access (★ cost 절감 + 보안).

---

## 13. image signing (★ Sigstore + ECR)

```bash
# cosign sign (keyless)
cosign sign --yes \
    123.dkr.ecr.ap-northeast-2.amazonaws.com/my-app@sha256:abc...

# verify
cosign verify \
    --certificate-identity-regexp "^https://github.com/myorg/.*" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    123.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:1.0
```

→ EKS admission webhook 가 signed image 만 허용.

---

## 14. GitHub Actions 통합 (★)

```yaml
name: build
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4

      # AWS Role assume (OIDC, no access key)
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/github-actions-ecr
          aws-region: ap-northeast-2

      # ECR login
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      # buildx
      - uses: docker/setup-buildx-action@v3

      # build + push + multi-arch + cache
      - uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ steps.ecr.outputs.registry }}/my-app:${{ github.sha }}
            ${{ steps.ecr.outputs.registry }}/my-app:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 15. EKS pull (★)

```yaml
# Pod 의 imagePullSecrets 안 필요 (IRSA / instance role 권한)
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      image: 123.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:1.0
```

→ node 의 IAM role 에 ECR pull policy 만 있으면 자동.

→ cross-account ECR 의 경우 repository policy 추가.

---

## 16. troubleshoot

```bash
# pull 실패
kubectl describe pod <name>
# Events 에서 ImagePullBackOff 원인

# IAM 권한 확인
aws sts get-caller-identity
aws ecr get-authorization-token   # 성공해야

# image 존재 확인
aws ecr describe-images --repository-name my-app

# scan 결과
aws ecr describe-image-scan-findings \
    --repository-name my-app \
    --image-id imageTag=1.0

# 옛 image 자동 삭제 확인
aws ecr describe-lifecycle-policy --repository-name my-app
```

---

## 17. cost 분석

```
1 TB image storage = $100/mo

reduce:
  - lifecycle policy (옛 image 삭제)
  - layer 재사용 (Dockerfile order)
  - 작은 base image (alpine / distroless)
  - multi-stage build
  - .dockerignore (불필요 파일 제외)

cross-region replication:
  - $0.10/GB (저장) × 2 region
  - + $0.02/GB (transfer)

pull-through cache:
  - upstream 의 비용 절감 (Docker Hub rate)
  - 자체 storage cost 추가
```

---

## 18. 함정

1. **tag mutable + same tag 덮어쓰기** — old pod 의 image 가 사라짐. IMMUTABLE.
2. **lifecycle 없음** — storage 무한 증가.
3. **NAT Gateway egress 무지** — VPC endpoint 필수.
4. **public ECR 와 혼동** — 권한 / 비용 다름.
5. **token 12h 만료** — long-running build 시 재login.
6. **cross-account 권한 없음** — repository policy 명시.
7. **image scan 안 봄** — Critical CVE 무시.
8. **multi-arch 없이 EKS** — ARM node 가 amd64 image fail.

---

## 19. 관련

- [[cloud-aws|↑ cloud-aws]]
- [[eks]]
- [[supply-chain-security|↗ supply chain]]
- [[buildkit-deep|↗ BuildKit]]
