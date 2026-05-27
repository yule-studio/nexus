---
title: "troubleshooting · ECS TaskFailedToStart / CannotPullContainerError"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, aws, ecs, fargate, vpc, ecr]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[../network/network]]"]
---

# ECS — `TaskFailedToStart` / `CannotPullContainerError`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
CannotPullContainerError: pull image manifest has been retried 5 time(s):
  failed to resolve ref ...
```

또는:
```
ResourceInitializationError: unable to pull secrets or registry auth:
  ... timeout
```

- ECS service 가 task 시작 실패 반복
- CloudWatch event 에 task STOPPED with reason

## 원인 (5가지 흔함)

1. **Network unreachable** — private subnet 에 NAT / VPC Endpoint 없어서 ECR 도달 못함
2. **IAM execution role 부족** — `AmazonECSTaskExecutionRolePolicy` 누락
3. **Image arch mismatch** — [[docker-arch-mismatch]]
4. **Secret arn 잘못** — Secrets Manager / SSM 경로 오타
5. **CPU/Memory 부족** — Fargate 할당 안 됨

## 해결

### Private subnet + ECR 도달
```
private subnet → NAT Gateway → Internet → ECR
또는
private subnet → VPC Endpoint (Interface) → ECR (AWS network)
```

VPC Endpoint 가 더 저렴 + 빠름. 필요한 endpoint:
- `com.amazonaws.<region>.ecr.api`
- `com.amazonaws.<region>.ecr.dkr`
- `com.amazonaws.<region>.s3` (Gateway endpoint — ECR layer)
- `com.amazonaws.<region>.logs`
- `com.amazonaws.<region>.secretsmanager`
- `com.amazonaws.<region>.ssm`

### Execution role
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
        "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "ssm:GetParameters", "kms:Decrypt"],
      "Resource": "arn:aws:secretsmanager:...:secret:myapp-*"
    }
  ]
}
```

→ AWS managed `AmazonECSTaskExecutionRolePolicy` + 추가 secret 권한.

### Task definition secrets
```json
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:123:secret:myapp-db-abc:password::"
    }
  ]
}
```

→ valueFrom 의 ARN + JSON key (`password`) 정확히.

### Image platform 명시
```json
{
  "runtimePlatform": {
    "cpuArchitecture": "ARM64",
    "operatingSystemFamily": "LINUX"
  }
}
```

### Health check 조정 (CrashLoopBackOff 와 혼동 X)
```json
{
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -fsS http://localhost:8080/health || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
}
```

## 진단

```bash
# Task 의 stop reason
aws ecs describe-tasks --cluster mycluster --tasks <task-arn>
# → stoppedReason 필드 확인

# Service 의 최근 event
aws ecs describe-services --cluster mycluster --services myservice \
  --query 'services[0].events[:10]'

# Container instance / Fargate 의 보안 그룹
aws ec2 describe-security-groups --group-ids sg-xxx

# VPC Endpoint 존재 확인
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-xxx

# Task definition 검증
aws ecs describe-task-definition --task-definition myapp:latest | jq '.taskDefinition'
```

## 재발 방지

- private subnet 에 production 띄울 때 VPC Endpoint 표준
- CDK / Terraform 으로 인프라 IaC — VPC Endpoint 누락 방지
- Task definition validator (CI)
- 모니터링: CloudWatch Events → Discord — task STOPPED 즉시 알람

## 관련

- [[devops]] / [[ecr-push-denied]] / [[../network/dns-resolve-fail]] / [[docker-arch-mismatch]]
