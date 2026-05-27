---
title: "ECS 04-1 — ECS Exec (컨테이너 shell)"
kind: knowledge
status: current
created_at: 2026-05-22T14:00:00+09:00
tags: [aws, ecs, operations, ecs-exec, ssm]
home_hub: ecs
---

# ECS Exec

> Fargate / EC2 task container 안 shell — SSM Agent + Session Manager. Fargate SSH 부재 해결.

## 1. 사용

```bash
aws ecs execute-command \
  --cluster bkurs-prod --task abc123... --container api \
  --command "/bin/bash" --interactive
```

## 2. 활성 조건 3

| 조건 | 설정 |
| --- | --- |
| Service 의 `enableExecuteCommand: true` | `aws ecs update-service --enable-execute-command` |
| Task Role 에 SSM Messages 권한 | 아래 |
| Cluster logging (권장) | `executeCommandConfiguration` |

### Task Role policy
```json
{
  "Effect": "Allow",
  "Action": [
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ],
  "Resource": "*"
}
```

### Cluster audit log
```bash
aws ecs update-cluster --cluster bkurs-prod \
  --configuration '{
    "executeCommandConfiguration": {
      "logging": "OVERRIDE",
      "logConfiguration": {
        "cloudWatchLogGroupName": "/ecs/exec-audit",
        "cloudWatchEncryptionEnabled": true
      }
    }
  }'
```

→ 모든 exec session 의 input/output 이 CloudWatch — audit.

## 3. 활성

```bash
# 신규
aws ecs create-service --enable-execute-command ...

# 기존
aws ecs update-service --cluster bkurs-prod --service api \
  --enable-execute-command --force-new-deployment
```

## 4. Session Manager Plugin

```bash
brew install --cask session-manager-plugin
```

## 5. 활용 — debug

```bash
TASK=$(aws ecs list-tasks --cluster bkurs-prod --service-name api --query 'taskArns[0]' --output text)

# 환경
aws ecs execute-command --cluster bkurs-prod --task $TASK --container api \
  --command "env" --interactive

# Spring Boot thread dump
aws ecs execute-command ... --command "jcmd 1 Thread.print" --interactive

# heap dump
aws ecs execute-command ... --command "jcmd 1 GC.heap_dump /tmp/heap.hprof" --interactive
```

## 6. audit

```bash
aws logs tail /ecs/exec-audit --follow
```

IAM 정책 — Developer tag 만 exec:
```json
{
  "Effect": "Allow",
  "Action": "ecs:ExecuteCommand",
  "Resource": "arn:aws:ecs:...:cluster/bkurs-prod",
  "Condition": {
    "StringEquals": { "aws:PrincipalTag/Role": "Developer" }
  }
}
```

## 7. kubectl exec 비교

| 항목 | ECS Exec | kubectl exec |
| --- | --- | --- |
| 인증 | IAM | K8s RBAC |
| audit | CloudWatch | kube-apiserver audit |
| port-forward | ✗ (SSM 별도) | ✓ |

## 8. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `enableExecuteCommand: false` | update + force-new-deployment |
| Task Role SSM 권한 누락 | ssmmessages:* 추가 |
| Plugin 미설치 | local 설치 |
| Distroless image (no shell) | netshoot sidecar |
| AWS CLI v1 | v2 권장 |

## 9. 권장 보안

- production: 활성하되 audit 필수
- IAM tag (Developer) 로 exec 권한 제한
- 모든 session log → SOC 검토

## 10. 다음

- ▶ [[02-container-insights]]
- ▶ [[03-troubleshooting]]
- ▶ [[../05-security/01-iam-best-practices]]

## 11. 영상 / 자료

| 자료 |
| --- |
| [ECS Exec — AWS](https://www.youtube.com/watch?v=zHfHwvFiy_E) |
| [공식 ECS Exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html) |

## 12. 참고

- [[../ecs]]
- [[../02-task-definition/06-iam-roles]]
