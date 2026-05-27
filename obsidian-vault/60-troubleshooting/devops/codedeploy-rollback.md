---
title: "troubleshooting · CodeDeploy Blue-Green rollback"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, aws, codedeploy, blue-green, alb]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[ecs-task-failed-to-start]]"]
---

# CodeDeploy Blue-Green deployment failed

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
The deployment failed because the test traffic listener did not register
all targets within the timeout period.
```

또는:
```
Health check failed: Target.Timeout
Deployment rolled back to previous version.
```

## 원인

1. Target Group health check fail (new task 가 unhealthy)
2. ALB health check path / port 잘못
3. CloudWatch alarm trigger (5xx / latency)
4. New task 자체 시작 실패 ([[ecs-task-failed-to-start]])
5. Termination wait 가 너무 짧음

## 해결

### Health check 정확히
```hcl
# Terraform
resource "aws_lb_target_group" "app" {
  name        = "app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    path                = "/actuator/health/readiness"   # liveness 와 분리
    port                = 8080
    interval            = 15
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200"
  }

  deregistration_delay = 30   # 빠른 deregister (느린 connection drain)
}
```

### Service grace period
```json
{
  "healthCheckGracePeriodSeconds": 120
}
```

→ Spring Boot 의 긴 startup 시간 흡수.

### Spring Boot Liveness / Readiness 분리
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate.enabled: true
    readinessstate.enabled: true
```

`/actuator/health/liveness` — 살아있는가
`/actuator/health/readiness` — 트래픽 받을 준비됨

ALB → readiness, K8s livenessProbe → liveness.

### Alarm-based rollback
```hcl
resource "aws_cloudwatch_metric_alarm" "high_5xx" {
  alarm_name          = "app-5xx-spike"
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 2
  threshold           = 10
  comparison_operator = "GreaterThanThreshold"
}
```

```bash
aws deploy update-deployment-group \
  --application-name myapp \
  --current-deployment-group-name dg1 \
  --alarm-configuration "enabled=true,alarms=[{name=app-5xx-spike}]"
```

→ alarm trigger 시 자동 rollback.

### Termination wait
```bash
aws deploy create-deployment-group \
  --blue-green-deployment-configuration '{
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 10   # 10분 wait — rollback 윈도우
    }
  }'
```

→ 새 버전 정상 확인 시간 충분히.

## 진단

```bash
# Deployment 상태
aws deploy get-deployment --deployment-id <id>

# Lifecycle events 의 실패한 hook
aws deploy get-deployment --deployment-id <id> \
  --query 'deploymentInfo.deploymentOverview'

# Target group health
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# ECS service event
aws ecs describe-services --cluster mycluster --services myservice \
  --query 'services[0].events[:10]'

# CloudWatch logs
aws logs tail /ecs/myapp --follow --since 30m
```

## 재발 방지

- Liveness / Readiness probe 분리
- Health check grace period = startup time + 30s 여유
- Alarm 기반 rollback 의무 (5xx + 응답시간 + task health)
- Linear deployment (10% / 1분씩) — 문제 빨리 감지

## 관련

- [[devops]] / [[ecs-task-failed-to-start]] / [[k8s-crashloopbackoff]]
