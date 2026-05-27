---
title: "ECR 실습 1 — Create + Push + Pull (buildx)"
kind: practice
status: current
created_at: 2026-05-22T17:54:00+09:00
tags: [aws, ecr, practice, docker, buildx]
home_hub: ecr
---

# 실습 1 — ECR Create + Push + Pull

> 목표: ECR repository 생성 → multi-arch image build + push → ECS task pull. 15분.

## 1. ECR Repository

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-2

aws ecr create-repository --repository-name lab-app \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE \
  --encryption-configuration "encryptionType=AES256"

# Lifecycle (untagged 7d expire)
aws ecr put-lifecycle-policy --repository-name lab-app \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Untagged 7d",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }]
  }'
```

## 2. Application

```bash
mkdir lab-app && cd lab-app

cat > Dockerfile <<'EOF'
# syntax=docker/dockerfile:1.7
FROM --platform=$BUILDPLATFORM node:20-alpine AS build
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app /app
USER node
EXPOSE 8080
CMD ["node","server.js"]
EOF

cat > server.js <<'EOF'
const http = require('http');
const server = http.createServer((req, res) => {
  res.end(`Hello from ${require('os').hostname()} (${process.arch})\n`);
});
server.listen(8080, () => console.log('listening 8080'));
EOF

cat > package.json <<'EOF'
{ "name": "lab", "version": "1.0.0" }
EOF
```

## 3. Login

```bash
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCT.dkr.ecr.$REGION.amazonaws.com
```

> Token valid 12 hours.

## 4. Build (single arch) + Push

```bash
docker build -t lab-app:v1 .

docker tag lab-app:v1 $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v1
docker push $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v1
```

## 5. Build Multi-arch + Push (buildx)

```bash
# Buildx setup
docker buildx create --name lab-builder --use --bootstrap

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v2 \
  --push .
```

## 6. 확인

```bash
# List
aws ecr list-images --repository-name lab-app
# imageTag: v1, v2

# Image Index (manifest list)
docker buildx imagetools inspect $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v2
# Name:      .../lab-app:v2
# Manifests:
#   Platform: linux/amd64
#   Platform: linux/arm64

# Scan 결과
aws ecr wait image-scan-complete --repository-name lab-app --image-id imageTag=v2
aws ecr describe-image-scan-findings --repository-name lab-app --image-id imageTag=v2 \
  --query 'imageScanFindings.findingSeverityCounts'
```

## 7. Pull (다른 머신 / ECS / EKS)

```bash
# Local
docker pull $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v2
docker run --rm -p 8080:8080 $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v2

# 다른 터미널
curl http://localhost:8080
# > Hello from ... (arm64 / amd64)
```

## 8. Immutable tag 확인

```bash
# 같은 tag 재push 시도
docker push $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v1
# > ImageAlreadyExistsException: Image with image-tag 'v1' already exists.

# 새 tag 로
docker tag lab-app:v1 $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v1.0.1
docker push $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app:v1.0.1
```

## 9. Digest 로 pull

```bash
DIGEST=$(aws ecr describe-images --repository-name lab-app --image-ids imageTag=v2 \
  --query 'imageDetails[0].imageDigest' --output text)

docker pull $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-app@$DIGEST
```

## 10. ECS Task Definition 활용

```json
{
  "family": "lab-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256", "memory": "512",
  "runtimePlatform": { "cpuArchitecture": "ARM64", "operatingSystemFamily": "LINUX" },
  "containerDefinitions": [{
    "name": "app",
    "image": "ACCT.dkr.ecr.ap-northeast-2.amazonaws.com/lab-app:v2",
    "essential": true,
    "portMappings": [{"containerPort": 8080}]
  }],
  "executionRoleArn": "arn:aws:iam::ACCT:role/ecsTaskExecutionRole"
}
```

`ecsTaskExecutionRole` 에 `AmazonEC2ContainerRegistryReadOnly` 부여 필수.

## 11. Clean up

```bash
aws ecr batch-delete-image --repository-name lab-app \
  --image-ids imageTag=v1 imageTag=v2 imageTag=v1.0.1
aws ecr delete-repository --repository-name lab-app --force
docker buildx rm lab-builder
```

## 12. 학습 포인트

- ECR repository (IMMUTABLE + scanOnPush)
- Lifecycle Policy 7d untagged
- buildx multi-arch (linux/amd64 + arm64)
- Image Index (manifest list)
- Digest pull (재현성)
- ECS Execution Role 의 ECR ReadOnly

## 13. 다음

- ▶ [[02-multi-arch-and-lifecycle]]
- ▶ [[../03-security/01-scanning-and-inspector]]
- ▶ [[../../ecs/practice/01-fargate-first-service]]

## 14. 참고

- [[../ecr]]
- [[../02-operations/01-push-pull-auth]]
- [[../../../docker/practice/06-multi-platform-build]]
