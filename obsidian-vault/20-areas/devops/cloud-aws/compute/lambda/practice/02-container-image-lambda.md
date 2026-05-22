---
title: "Lambda 실습 2 — Container Image + ECR + Spring Boot native"
kind: practice
status: current
created_at: 2026-05-22T17:26:00+09:00
tags: [aws, lambda, practice, container, ecr, spring-boot, native]
home_hub: lambda
---

# 실습 2 — Container Image Lambda

> 목표: GraalVM native + Spring Boot → Lambda container image 10MB. 40분.

## 1. ECR repository

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-2

aws ecr create-repository --repository-name lab-fn \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE
```

## 2. Spring Boot native (GraalVM)

```bash
# build.gradle.kts
# plugins {
#   id("org.springframework.boot") version "3.5.0"
#   id("org.graalvm.buildtools.native") version "0.10.2"
# }
# dependencies {
#   implementation("org.springframework.cloud:spring-cloud-function-adapter-aws:4.1.0")
# }

./gradlew nativeCompile
# build/native/nativeCompile/lab-fn  (binary)
```

## 3. Dockerfile (Lambda container)

```dockerfile
# Multi-stage native compile (Java 21 + GraalVM)
FROM ghcr.io/graalvm/native-image-community:21 AS build
WORKDIR /src
COPY --link gradlew settings.gradle.kts build.gradle.kts ./
COPY --link gradle gradle
RUN ./gradlew --no-daemon dependencies || true
COPY --link . .
RUN ./gradlew --no-daemon nativeCompile

# Final image (Lambda Runtime API)
FROM public.ecr.aws/lambda/provided:al2023-arm64
COPY --from=build /src/build/native/nativeCompile/lab-fn ${LAMBDA_RUNTIME_DIR}/bootstrap
RUN chmod +x ${LAMBDA_RUNTIME_DIR}/bootstrap
CMD ["lab-fn.Handler"]
```

> Python 의 경우 `public.ecr.aws/lambda/python:3.12` base image.

```dockerfile
# Python container 예
FROM public.ecr.aws/lambda/python:3.12

COPY requirements.txt .
RUN pip install -r requirements.txt -t ${LAMBDA_TASK_ROOT}
COPY app.py ${LAMBDA_TASK_ROOT}

CMD ["app.lambda_handler"]
```

## 4. Image build + push

```bash
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCT.dkr.ecr.$REGION.amazonaws.com

docker buildx build --platform linux/arm64 \
  -t $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-fn:v1 --push .
```

## 5. Lambda 생성 (container)

```bash
aws iam create-role --role-name LabContainerRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy --role-name LabContainerRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws lambda create-function --function-name lab-container \
  --package-type Image \
  --code ImageUri=$ACCT.dkr.ecr.$REGION.amazonaws.com/lab-fn:v1 \
  --role arn:aws:iam::$ACCT:role/LabContainerRole \
  --architectures arm64 \
  --memory-size 1024 --timeout 60 \
  --tracing-config Mode=Active
```

## 6. 호출

```bash
aws lambda invoke --function-name lab-container \
  --payload '{"name":"world"}' --cli-binary-format raw-in-base64-out \
  out.json
cat out.json
```

## 7. Function URL (HTTP test)

```bash
aws lambda create-function-url-config --function-name lab-container \
  --auth-type NONE \
  --cors '{
    "AllowOrigins":["*"],"AllowMethods":["*"],"AllowHeaders":["*"]
  }'

URL=$(aws lambda get-function-url-config --function-name lab-container \
  --query FunctionUrl --output text)
curl "$URL"
```

## 8. Cold start 측정

```bash
aws lambda invoke --function-name lab-container out.json
aws lambda invoke --function-name lab-container out.json   # warm

# CloudWatch Logs
aws logs filter-log-events --log-group-name /aws/lambda/lab-container \
  --filter-pattern "REPORT" --max-items 5
```

전형적:
```
REPORT ... Duration: 80ms Init Duration: 250ms     ← cold start
REPORT ... Duration: 8ms                            ← warm
```

> Java 일반 cold start ~3000ms → GraalVM native ~250ms.

## 9. Image update — version

```bash
# 새 image push
docker buildx build --platform linux/arm64 \
  -t $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-fn:v2 --push .

# Lambda update
aws lambda update-function-code --function-name lab-container \
  --image-uri $ACCT.dkr.ecr.$REGION.amazonaws.com/lab-fn:v2 \
  --publish

# Alias
aws lambda create-alias --function-name lab-container \
  --name prod --function-version 2
```

## 10. CDK 버전

```typescript
new lambda.DockerImageFunction(this, 'Fn', {
  code: lambda.DockerImageCode.fromImageAsset('./fn'),
  memorySize: 1024,
  timeout: cdk.Duration.seconds(60),
  architecture: lambda.Architecture.ARM_64,
  tracing: lambda.Tracing.ACTIVE,
});
```

## 11. Image scan + immutable

ECR 의 EnhanceScanning + IMMUTABLE tag — 이전 image 변경 금지.

```bash
aws inspector2 enable --resource-types ECR
aws ecr put-image-tag-mutability --repository-name lab-fn \
  --image-tag-mutability IMMUTABLE
```

## 12. Cold start 단축 — 추가

| 방법 | 효과 |
| --- | --- |
| GraalVM native (Java) | -90% |
| SnapStart (Java JVM) | -85% |
| Provisioned Concurrency | 0 |
| arm64 | -10~20% |
| 작은 image (multi-stage) | image pull ↓ |
| Image caching layer | -50% (warm host) |

## 13. Clean up

```bash
aws lambda delete-function --function-name lab-container
aws iam detach-role-policy --role-name LabContainerRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name LabContainerRole
aws ecr delete-repository --repository-name lab-fn --force
aws logs delete-log-group --log-group-name /aws/lambda/lab-container
```

## 14. 학습 포인트

- Container image (10GB 한도) 활용
- Lambda Runtime API (`/var/runtime/bootstrap`)
- multi-stage Docker — image 크기 ↓
- GraalVM native = Java cold start 1/10
- Function URL — public HTTP endpoint 즉시
- Image immutability + scan

## 15. 다음

- ▶ [[03-event-driven-pipeline]]
- ▶ [[../01-fundamentals/02-runtime-and-deployment]]
- ▶ [[../../ecr/ecr]]

## 16. 참고

- [[../lambda]]
- [[../01-fundamentals/02-runtime-and-deployment]]
- [[../../ecr/ecr]]
