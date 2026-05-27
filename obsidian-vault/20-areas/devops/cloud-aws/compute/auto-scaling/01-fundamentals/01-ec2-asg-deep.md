---
title: "Auto Scaling 01-1 — EC2 ASG Deep (Launch Template / Mixed Instances)"
kind: knowledge
status: current
created_at: 2026-05-22T18:02:00+09:00
tags: [aws, asg, ec2, launch-template, mixed-instances, spot]
home_hub: auto-scaling
---

# EC2 Auto Scaling Group (ASG) Deep

> Launch Template + min/desired/max + Mixed Instances Policy + Health Check + Lifecycle. Capacity Provider (ECS) / EKS Cluster Autoscaler 의 토대.

## 1. ASG 의 구성

```mermaid
flowchart LR
  LT[Launch Template]
  POL[Mixed Instances Policy]
  ASG[Auto Scaling Group<br/>min / desired / max]
  TG[ALB Target Group]
  HC[Health Check<br/>EC2 / ELB]
  SCH[Scaling Policy]
  EC2[EC2 instances]

  LT --> ASG
  POL --> ASG
  TG -.|register| ASG
  HC -.|monitor| ASG
  SCH -.|adjust| ASG
  ASG --> EC2
```

## 2. Launch Template

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

aws ec2 create-launch-template --launch-template-name lab-lt \
  --version-description v1 \
  --launch-template-data "{
    \"ImageId\": \"$AMI\",
    \"InstanceType\": \"t4g.small\",
    \"IamInstanceProfile\": { \"Name\": \"EC2SSMProfile\" },
    \"SecurityGroupIds\": [\"sg-xxx\"],
    \"UserData\": \"$(base64 -i user-data.sh)\",
    \"MetadataOptions\": { \"HttpTokens\": \"required\", \"HttpPutResponseHopLimit\": 2 },
    \"BlockDeviceMappings\": [{
      \"DeviceName\": \"/dev/xvda\",
      \"Ebs\": { \"VolumeSize\": 30, \"VolumeType\": \"gp3\", \"Encrypted\": true, \"DeleteOnTermination\": true }
    }],
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{ \"Key\": \"Name\", \"Value\": \"lab-asg\" }]
    }]
  }"

# Version 추가
aws ec2 create-launch-template-version --launch-template-name lab-lt \
  --source-version 1 \
  --launch-template-data "{ \"InstanceType\": \"t4g.medium\" }"
```

ASG 의 reference:
```
LaunchTemplateName: lab-lt
Version:            $Latest / $Default / 1 / 2 / ...
```

## 3. ASG 생성

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name lab-asg \
  --launch-template "LaunchTemplateName=lab-lt,Version=\$Latest" \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-a,subnet-b,subnet-c" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/lab/... \
  --health-check-type ELB --health-check-grace-period 120 \
  --default-cooldown 60 \
  --capacity-rebalance \
  --tags ResourceId=lab-asg,ResourceType=auto-scaling-group,Key=Environment,Value=prod,PropagateAtLaunch=true
```

| 옵션 | 의미 |
| --- | --- |
| `min-size` / `desired-capacity` / `max-size` | 한도 |
| `vpc-zone-identifier` | subnet (AZ 자동 spread) |
| `target-group-arns` | ALB / NLB target group |
| `health-check-type` | EC2 (default) / ELB |
| `health-check-grace-period` | bootup 동안 unhealthy 무시 |
| `default-cooldown` | scaling 사이 대기 (60-300s) |
| `capacity-rebalance` | Spot interruption 대비 미리 교체 |

## 4. Health Check

### EC2 (default)
- EC2 Status Check (Instance + System)
- failed 시 instance 교체

### ELB
- target group health check 통합
- 더 정확 (application level)
- 권장 — production

### Custom
```bash
aws autoscaling set-instance-health --instance-id i-xxx \
  --health-status Unhealthy
```

→ application 자체 health 결정.

## 5. Mixed Instances Policy

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-asg \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "lab-lt", "Version": "$Latest"
      },
      "Overrides": [
        { "InstanceRequirements": {
          "VCpuCount": { "Min": 2, "Max": 4 },
          "MemoryMiB": { "Min": 4096, "Max": 16384 },
          "CpuManufacturers": ["amazon-web-services"],
          "InstanceGenerations": ["current"]
        }}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "capacity-optimized",
      "SpotInstancePools": 4
    }
  }'
```

### Instance Requirements (Attribute-based selection)
- VCpu / Memory / CPU manufacturer / generation
- 자동 매칭 (m7g.large / m6g.large / m6gd.large 등 자동)

### InstancesDistribution
- `OnDemandBaseCapacity` — 보장 On-Demand 수
- `OnDemandPercentageAboveBaseCapacity` — base 위 OD %
- `SpotAllocationStrategy`:
  - `capacity-optimized` ★ (interruption 가장 적음)
  - `capacity-optimized-prioritized`
  - `price-capacity-optimized` (2022+, 권장)
  - `lowest-price` (interruption 위험)

## 6. Spot Interruption 대응

```yaml
capacity-rebalance: true       # 2분 통지 전에 미리 교체
```

Lifecycle Hook 으로 graceful:
```bash
aws autoscaling put-lifecycle-hook --auto-scaling-group-name lab-asg \
  --lifecycle-hook-name spot-drain \
  --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --notification-target-arn arn:sqs:... \
  --role-arn arn:iam::...:role/asg-hook
```

→ Spot 회수 직전 SQS message → graceful drain script.

## 7. AZ Balancing

```
ASG 자동 spread:
- 3 AZ × desired 9 = 3 / 3 / 3
- AZ 한 곳 down → 다른 AZ 에서 자동 보충
- Cross-AZ rebalance 자동
```

## 8. Suspend / Resume Process

```bash
aws autoscaling suspend-processes --auto-scaling-group-name lab-asg \
  --scaling-processes HealthCheck Terminate

# 모든 process
aws autoscaling suspend-processes --auto-scaling-group-name lab-asg
```

| Process | 의미 |
| --- | --- |
| Launch | 새 instance 시작 |
| Terminate | 기존 종료 |
| HealthCheck | health check |
| ReplaceUnhealthy | unhealthy 자동 교체 |
| ScheduledActions | 스케줄 |
| AlarmNotification | metric trigger |
| AZRebalance | AZ 균등 |
| AddToLoadBalancer | TG 등록 |

→ debugging / maintenance 시 suspend.

## 9. Instance Refresh (Rolling Replace)

```bash
aws autoscaling start-instance-refresh --auto-scaling-group-name lab-asg \
  --strategy Rolling \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 60,
    "MaxHealthyPercentage": 110,
    "SkipMatching": true
  }'

# 진행
aws autoscaling describe-instance-refreshes --auto-scaling-group-name lab-asg \
  --query 'InstanceRefreshes[0].{Status:Status,Percentage:PercentageComplete}'

# Rollback
aws autoscaling rollback-instance-refresh --auto-scaling-group-name lab-asg
```

→ Launch Template 변경 후 zero-downtime rolling.

## 10. Termination Policy

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name lab-asg \
  --termination-policies "OldestLaunchTemplate" "AllocationStrategy" "Default"
```

옵션:
- `OldestInstance` — 가장 오래된 instance
- `NewestInstance` — 가장 새 (rollback 시)
- `OldestLaunchTemplate` — old version 부터 (Rolling Refresh 와 결합)
- `OldestLaunchConfiguration`
- `AllocationStrategy` — Spot allocation 유지
- `Default` — AZ balance 후 oldest

## 11. ASG 한도

| | |
| --- | --- |
| ASG / account / region | 500 (soft) |
| Launch Template / region | 5000 |
| Launch Template versions | 10000 / template |
| Mixed Instance Overrides | 40 |
| Lifecycle Hooks / ASG | 50 |

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| desired-capacity 직접 변경 → policy 가 다시 변경 | scaling policy 만 사용 |
| ELB health check 미설정 → unhealthy 모름 | --health-check-type ELB |
| grace period 짧음 → 부팅 중 죽음 | 120+ |
| Spot allocation lowest-price → 자주 죽음 | capacity-optimized |
| Capacity Rebalance OFF → Spot 회수 직전 down | true |
| Termination Policy 의도 안 맞음 | OldestLaunchTemplate |

## 13. 다음

- ▶ [[02-application-auto-scaling]]
- ▶ [[03-predictive-and-target-tracking]]
- ▶ [[../03-operations/01-lifecycle-hooks-and-warm-pool]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [ASG 공식](https://docs.aws.amazon.com/autoscaling/ec2/userguide/) |
| [Spot best practices](https://docs.aws.amazon.com/autoscaling/ec2/userguide/asg-using-spot-instances.html) |

## 15. 참고

- [[../auto-scaling]]
- [[../../ec2/01-fundamentals/03-pricing-models]]
- [[../../ec2/practice/03-asg-with-launch-template]]
