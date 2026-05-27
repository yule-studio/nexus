---
title: "ECS 02-5 — Environment / Secrets (SSM / Secrets Manager / envFiles)"
kind: knowledge
status: current
created_at: 2026-05-22T13:38:00+09:00
tags: [aws, ecs, task-definition, env, secrets, ssm, secrets-manager]
home_hub: ecs
---

# Task Definition — Environment / Secrets

> 컨테이너 안 env 변수 주입 3 방법: `environment` (plain) / `secrets` (SSM / Secrets Manager) /
> `environmentFiles` (S3). SSM Parameter Store 가 cost-effective, Secrets Manager 는 rotation 지원.

## 1. 3 방법 비교

| 방법 | 비용 | 보안 | 용도 |
| --- | --- | --- | --- |
| **environment** | 무료 | 평문 (task def 에 노출) | non-secret config |
| **secrets** (SSM / Secrets Manager) | SSM ~무료 / SM $0.40/secret/월 | KMS 암호화 | DB pwd / API key |
| **environmentFiles** (S3) | S3 비용 | 평문 (S3 객체) | 대량 plain env |

## 2. environment — plain

```json
{
  "environment": [
    { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
    { "name": "JAVA_OPTS", "value": "-XX:MaxRAMPercentage=75" },
    { "name": "AWS_REGION", "value": "ap-northeast-2" }
  ]
}
```

특징:
- task definition 에 **평문 저장** (IAM `ecs:DescribeTaskDefinition` 권한자 모두 볼 수 있음)
- 무료, 변경 = 새 revision

## 3. secrets — SSM Parameter Store (권장)

```json
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:ssm:ap-northeast-2:123:parameter/bkurs-be/prod/DB_PASSWORD"
    },
    {
      "name": "JWT_SECRET",
      "valueFrom": "arn:aws:ssm:ap-northeast-2:123:parameter/bkurs-be/prod/JWT_SECRET"
    }
  ]
}
```

### Parameter 생성

```bash
aws ssm put-parameter \
  --name /bkurs-be/prod/DB_PASSWORD \
  --type SecureString \
  --value 'mySecret!' \
  --key-id alias/aws/ssm
```

### Task Execution Role 권한

```json
{ "Effect": "Allow", "Action": ["ssm:GetParameters"], "Resource": "arn:aws:ssm:...:parameter/bkurs-be/prod/*" },
{ "Effect": "Allow", "Action": ["kms:Decrypt"], "Resource": "arn:aws:kms:...:key/aws/ssm" }
```

## 4. secrets — AWS Secrets Manager

자동 rotation / cross-region replication 지원. 비싸지만 강력.

```json
{
  "secrets": [{
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:...:secret:bkurs-be/prod/db-AbCdEf:password::"
  }]
}
```

ARN suffix:
- `:password::` — JSON 의 `password` key 만
- `::` — secret 전체

### 자동 rotation

```bash
aws secretsmanager rotate-secret \
  --secret-id bkurs-be/prod/db \
  --rotation-lambda-arn arn:aws:lambda:...:function:SecretsManagerRDSPostgreSQLRotationSingleUser \
  --rotation-rules AutomaticallyAfterDays=30
```

## 5. SSM vs Secrets Manager — 결정

| 항목 | SSM | Secrets Manager |
| --- | --- | --- |
| 가격 | standard 무료 | $0.40/secret/월 + $0.05/10k call |
| 자동 rotation | ✗ | ✓ |
| 큰 값 | 8KB | 64KB |
| 권장 | 일반 secret | rotation 필수 |

> bkurs-be: **SSM Parameter Store SecureString**.

## 6. environmentFiles — S3

```json
{
  "environmentFiles": [{
    "value": "arn:aws:s3:::bkurs-config/env/prod.env",
    "type": "s3"
  }]
}
```

prod.env:
```
ENV=prod
LOG_LEVEL=info
```

Task Execution Role 에 `s3:GetObject` 권한 필요. 권장: 평범한 plain env 50+ 일 때만.

## 7. 권장 조합 — bkurs-be

```json
{
  "containerDefinitions": [{
    "name": "api",
    "environment": [
      { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" },
      { "name": "DB_HOST", "value": "db.bkurs-prod.internal" },
      { "name": "AWS_REGION", "value": "ap-northeast-2" }
    ],
    "secrets": [
      { "name": "DB_PASSWORD",        "valueFrom": "arn:aws:ssm:...:parameter/bkurs-be/prod/DB_PASSWORD" },
      { "name": "JWT_SECRET",         "valueFrom": "arn:aws:ssm:...:parameter/bkurs-be/prod/JWT_SECRET" },
      { "name": "ANTHROPIC_API_KEY",  "valueFrom": "arn:aws:ssm:...:parameter/bkurs-be/prod/ANTHROPIC_API_KEY" },
      { "name": "PORTONE_IMP_SECRET", "valueFrom": "arn:aws:ssm:...:parameter/bkurs-be/prod/PORTONE_IMP_SECRET" }
    ]
  }]
}
```

자세히 → [[../../../../10-projects/bkurs-be/handover/operations/env-deployment#5]]

## 8. credential 주입 흐름

```
ECS agent 가 task 시작 시 SSM / Secrets Manager 호출 (Execution Role 권한)
→ 응답값을 컨테이너 env 로 주입
→ 컨테이너 안 앱은 평소처럼 `System.getenv("DB_PASSWORD")`
```

→ 코드 측 변경 없음 — SSM/SM 통합은 ECS 가 처리.

## 9. CDK 예

```typescript
container.addContainer('api', {
  secrets: {
    DB_PASSWORD: ecs.Secret.fromSsmParameter(
      ssm.StringParameter.fromSecureStringParameterAttributes(this, 'P', {
        parameterName: '/bkurs-be/prod/DB_PASSWORD',
      })
    ),
  },
});
```

## 10. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `environment` 에 secret | `secrets` 사용 |
| Execution Role 권한 부족 | SSM + KMS Decrypt |
| KMS default vs customer-managed | production 권장 customer-managed |
| secret 변경 후 안 적용 | `--force-new-deployment` |
| log 에 env dump | logger mask + level |

## 11. 다음

- ▶ [[06-iam-roles]] — Execution Role 권한 세팅
- ▶ [[07-logging]]
- ▶ [[../05-security/03-secret-management]] — rotation / KMS
- ▶ [[../../security/secrets-manager/secrets-manager]] — Secrets Manager 깊이

## 12. 영상 / 자료

| 자료 |
| --- |
| [ECS Secrets — AWS](https://www.youtube.com/watch?v=dt0M2chMI-c) |
| [SSM vs Secrets Manager](https://www.youtube.com/watch?v=yEcRR4y2sB8) |

## 13. 참고

- [[../ecs]]
- 공식: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data.html
