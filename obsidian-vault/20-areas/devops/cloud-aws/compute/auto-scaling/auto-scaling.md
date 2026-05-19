---
title: "Auto Scaling — ASG / Launch Template / Predictive"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:05:00+09:00
tags: [devops, cloud-aws, auto-scaling, asg]
---

# Auto Scaling — ASG / Launch Template / Predictive

**[[cloud-aws|↑ cloud-aws]]**

---

## 1. 무엇

```
Auto Scaling Group (ASG):
  - EC2 instance 의 desired / min / max 관리
  - health check fail → 자동 교체
  - traffic 변동 → scale-out / scale-in
  - multi-AZ 자동 분배

구성요소:
  - Launch Template (★ Launch Configuration 의 후속)
  - Scaling Policy
  - CloudWatch metric / alarm
```

---

## 2. Launch Template (★)

```bash
aws ec2 create-launch-template \
    --launch-template-name web-template \
    --version-description "v1" \
    --launch-template-data '{
        "ImageId": "ami-0abc123",
        "InstanceType": "m6i.large",
        "KeyName": "prod-key",
        "SecurityGroupIds": ["sg-xxx"],
        "IamInstanceProfile": {"Name": "EC2WebRole"},
        "UserData": "BASE64_ENCODED_SCRIPT",
        "Monitoring": {"Enabled": true},
        "TagSpecifications": [{
            "ResourceType": "instance",
            "Tags": [
                {"Key": "Name", "Value": "web-prod"},
                {"Key": "Environment", "Value": "prod"}
            ]
        }],
        "BlockDeviceMappings": [{
            "DeviceName": "/dev/xvda",
            "Ebs": {
                "VolumeSize": 30,
                "VolumeType": "gp3",
                "Encrypted": true,
                "DeleteOnTermination": true
            }
        }],
        "MetadataOptions": {
            "HttpTokens": "required",         # IMDSv2 강제 ★
            "HttpPutResponseHopLimit": 1
        }
    }'
```

→ Launch Configuration (legacy) X. Launch Template (★) 권장.

---

## 3. ASG 생성

```bash
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name web-asg \
    --launch-template "LaunchTemplateName=web-template,Version=1" \
    --min-size 2 \
    --max-size 20 \
    --desired-capacity 4 \
    --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc" \
    --target-group-arns "arn:aws:elasticloadbalancing:..." \
    --health-check-type ELB \
    --health-check-grace-period 300
```

→ `health-check-type ELB` = ELB health check 사용 (★ 권장). `EC2` 만 = ELB health 모름.

---

## 4. Mixed Instance Policy (★ Spot + On-Demand)

```bash
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name web-mixed \
    --mixed-instances-policy '{
        "LaunchTemplate": {
            "LaunchTemplateSpecification": {
                "LaunchTemplateName": "web-template",
                "Version": "1"
            },
            "Overrides": [
                {"InstanceType": "m6i.large"},
                {"InstanceType": "m5.large"},
                {"InstanceType": "m5a.large"},
                {"InstanceType": "m6a.large"},
                {"InstanceType": "m6i.xlarge"}
            ]
        },
        "InstancesDistribution": {
            "OnDemandPercentageAboveBaseCapacity": 30,
            "OnDemandBaseCapacity": 2,
            "SpotAllocationStrategy": "capacity-optimized",
            "SpotInstancePools": 5
        }
    }' \
    --min-size 2 --max-size 20 --desired-capacity 5
```

→ 2 on-demand base + 30% on-demand + 70% spot.

→ Spot 가용성 ↑ (여러 instance type / pool).

---

## 5. Scaling Policy 종류

### A. Target Tracking (★ 권장)

```bash
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name web-asg \
    --policy-name target-tracking-cpu \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration '{
        "TargetValue": 60.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ASGAverageCPUUtilization"
        }
    }'
```

→ 가장 단순. "CPU 60% 유지" → 자동 scale.

### B. Step Scaling

```bash
# CPU 80% over: +2 instance, 90% over: +5
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name web-asg \
    --policy-name step-scale-up \
    --policy-type StepScaling \
    --adjustment-type ChangeInCapacity \
    --step-adjustments '[
        {"MetricIntervalLowerBound": 0, "MetricIntervalUpperBound": 10, "ScalingAdjustment": 2},
        {"MetricIntervalLowerBound": 10, "ScalingAdjustment": 5}
    ]'
```

→ 세밀 control. CloudWatch alarm 과 연결 필요.

### C. Predictive Scaling (★ ML)

```bash
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name web-asg \
    --policy-name predictive-cpu \
    --policy-type PredictiveScaling \
    --predictive-scaling-configuration '{
        "MetricSpecifications": [{
            "TargetValue": 60.0,
            "PredefinedMetricPairSpecification": {
                "PredefinedMetricType": "ASGCPUUtilization"
            }
        }],
        "Mode": "ForecastAndScale",
        "SchedulingBufferTime": 300
    }'
```

→ AWS 가 24시간 trend 학습 → 미리 scale. 일정 pattern 의 traffic.

### D. Scheduled

```bash
aws autoscaling put-scheduled-update-group-action \
    --auto-scaling-group-name web-asg \
    --scheduled-action-name peak-hour-up \
    --start-time "2026-05-15T08:00:00Z" \
    --recurrence "0 8 * * 1-5" \
    --min-size 5 --max-size 30 --desired-capacity 10
```

→ 평일 8시 = 10 instance, 19시 = 2 instance.

---

## 6. health check

```
2 가지 type:
  EC2 (default): instance 의 status check
                 stopped / impaired = unhealthy

  ELB: ELB target group 의 health check
       application 응답 검증 (★ 권장)

grace period (★):
  새 instance 시작 후 N 초 = health check 무시
  → JVM cold start 등 고려
  → default 300s
```

```yaml
# Launch Template 의 user-data 가 끝날 때까지 충분히
HealthCheckGracePeriod: 600        # 10 min
```

---

## 7. termination policy

```
ASG scale-in 시 어느 instance 가 죽을지:

OldestInstance       — 가장 오래된
NewestInstance       — 가장 최근
OldestLaunchTemplate — 옛 Launch Template version
ClosestToNextInstanceHour — 청구 시간 가까운 (legacy)
AllocationStrategy   — Spot allocation 따라

default: AllocationStrategy → OldestLaunchConfiguration → ClosestToNextInstanceHour → Default
```

```bash
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name web-asg \
    --termination-policies "OldestLaunchTemplate"
```

→ 새 Launch Template version 으로 rolling update 시 OldestLaunchTemplate.

---

## 8. instance refresh (★ rolling update)

```bash
# Launch Template 변경 후 rolling replace
aws autoscaling start-instance-refresh \
    --auto-scaling-group-name web-asg \
    --preferences '{
        "MinHealthyPercentage": 80,
        "InstanceWarmup": 300,
        "CheckpointPercentages": [50, 100],
        "CheckpointDelay": 600
    }' \
    --strategy Rolling
```

→ AMI / instance type 변경 시 사용.

→ Blue-Green vs Rolling 선택.

---

## 9. lifecycle hook (★)

```
instance 시작 / 종료 시 잠시 stop:
  - 시작: load test 시 잠시 hold
  - 종료: graceful shutdown 시간 확보 (DB drain, cache flush)
```

```bash
aws autoscaling put-lifecycle-hook \
    --auto-scaling-group-name web-asg \
    --lifecycle-hook-name web-terminate-hook \
    --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
    --heartbeat-timeout 300 \
    --notification-target-arn arn:aws:sqs:... \
    --role-arn arn:aws:iam::...
```

```bash
# instance 안에서 (graceful shutdown)
# 1. lifecycle hook event 받음 (SNS / SQS / EventBridge)
# 2. graceful shutdown 시작 (DB connection close, ALB deregister)
# 3. complete signal
aws autoscaling complete-lifecycle-action \
    --lifecycle-action-result CONTINUE \
    --instance-id i-xxx \
    --lifecycle-hook-name web-terminate-hook \
    --auto-scaling-group-name web-asg
```

---

## 10. warm pool (★ 빠른 scale-out)

```bash
aws autoscaling put-warm-pool \
    --auto-scaling-group-name web-asg \
    --max-group-prepared-capacity 20 \
    --min-size 5 \
    --pool-state Stopped
```

→ stopped instance 미리 준비 → 필요 시 1분 안에 시작.

→ JVM cold start 5-10분 → 1분.

→ 비용: stopped instance 의 EBS 비용만.

---

## 11. cooldown

```
scale 후 wait time:
  - 너무 짧음 = flap (oscillation)
  - 너무 김 = slow response

target tracking: 자동 (default 300s)
step scaling: manual

```bash
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name web-asg \
    --cooldown 300       # 5 min
```

---

## 12. ELB target group 통합

```
ALB / NLB 의 target group ↔ ASG:
  - ASG 의 instance 가 자동 register
  - 종료 시 deregister
  - health check fail → 자동 교체
  - deregistration delay (★ 600s 권장)
```

```yaml
# Terraform
resource "aws_autoscaling_attachment" "web" {
  autoscaling_group_name = aws_autoscaling_group.web.name
  lb_target_group_arn    = aws_lb_target_group.web.arn
}
```

---

## 13. CloudWatch metric (필수)

```
metric:
  - CPU / memory / network (instance 별)
  - GroupDesiredCapacity
  - GroupInServiceInstances
  - GroupPendingInstances
  - GroupTerminatingInstances
  - ALB target 의 healthy host count
  - request count per target

alarm:
  - CPU > 80% for 5 min → scale-out
  - healthy host < 50% → page
  - failed launch count
```

---

## 14. spot 안정 전략

```
spot 의 갑작스 종료:

1. capacity-optimized strategy
   → 여러 instance type / AZ 의 가용 spot pool 우선

2. mixed instance policy
   → 5+ instance type
   → 2+ AZ

3. spot interruption handler
   → aws-node-termination-handler (k8s)
   → 또는 SNS + Lambda + custom

4. on-demand base capacity
   → critical 일부는 on-demand

5. graceful drain
   → ASG lifecycle hook
   → 2분 안에 정리
```

---

## 15. 활용 사례

### A. Web tier (ALB + ASG)

```
ALB → Target Group → ASG (4-20 web instance)
                       ↓
                     RDS / Cache

policy: target tracking CPU 60%
schedule: 평일 8-19 desired 8, off 4
```

### B. Worker (배치 / Queue consumer)

```
SQS queue depth → CloudWatch metric → ASG scale

policy: target tracking ApproximateNumberOfMessagesVisible 10
        = 10 message 당 1 instance
mixed: 90% Spot
```

### C. Bastion / Jump

```
ASG min=1 max=1
desired=1, 24/7
→ instance fail 시 자동 교체
→ AMI baked (SSH key / fail2ban)
```

---

## 16. 함정

1. **Launch Configuration 사용** — legacy. Launch Template 으로.
2. **EC2 health check 만** — application fail 인지 X. ELB.
3. **grace period 너무 짧음** — startup 중 healthy 판정 fail.
4. **deregistration delay 너무 짧음** — in-flight request 끊김.
5. **cooldown 너무 짧음** — flap.
6. **Spot 단일 type** — capacity fail.
7. **IMDSv1 default** — SSRF 공격 위험. v2 강제.
8. **AMI 의 secret hard-code** — IAM Role + Secrets Manager 사용.
9. **scaling metric 잘못** — CPU 만, 실제는 memory bound.
10. **warm pool 의 EBS 비용 무지** — 큰 EBS = stopped 도 부담.

---

## 17. 관련

- [[cloud-aws|↑ cloud-aws]]
- [[ec2]]
- [[eks]]
- [[alb|↗ ALB]]
- [[right-sizing|↗ right-sizing]]
