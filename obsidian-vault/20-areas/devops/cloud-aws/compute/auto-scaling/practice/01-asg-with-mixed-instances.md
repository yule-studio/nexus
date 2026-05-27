---
title: "Auto Scaling 실습 1 — ASG + Mixed (Spot+OD) + Target Tracking + Predictive + Warm Pool"
kind: practice
status: current
created_at: 2026-05-22T18:22:00+09:00
tags: [aws, asg, practice, spot, predictive, warm-pool]
home_hub: auto-scaling
---

# 실습 1 — ASG + Mixed Instances + Target Tracking + Predictive

> 목표: ALB + Launch Template + Mixed Spot/OD + Target Tracking CPU + Predictive + Warm Pool. 60분.

## 1. 인프라

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC \
  --query 'Subnets[].SubnetId' --output text))

# SG
SG_ALB=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-asg-alb \
  --description "alb" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0

SG_INST=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-asg-inst \
  --description "instance" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_INST --protocol tcp --port 80 --source-group $SG_ALB

# ALB
ALB_ARN=$(aws elbv2 create-load-balancer --name lab-asg-alb --type application \
  --subnets ${SUBNETS[@]} --security-groups $SG_ALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG_ARN=$(aws elbv2 create-target-group --name lab-asg-tg --protocol HTTP --port 80 \
  --vpc-id $VPC --target-type instance --health-check-path /health \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

## 2. Launch Template

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

cat > user-data.sh <<'EOF'
#!/bin/bash
dnf install -y nginx stress
cat > /etc/nginx/conf.d/default.conf <<'EOC'
server {
  listen 80;
  location /health { return 200 'ok'; add_header Content-Type text/plain; }
  location / { return 200 'host: $(hostname)\n'; add_header Content-Type text/plain; }
}
EOC
systemctl enable --now nginx
EOF

USER_DATA=$(base64 -i user-data.sh)

aws ec2 create-launch-template --launch-template-name lab-mixed-lt \
  --launch-template-data "{
    \"ImageId\": \"$AMI\",
    \"InstanceType\": \"t4g.small\",
    \"SecurityGroupIds\": [\"$SG_INST\"],
    \"UserData\": \"$USER_DATA\",
    \"MetadataOptions\": { \"HttpTokens\": \"required\", \"HttpPutResponseHopLimit\": 2 },
    \"BlockDeviceMappings\": [{
      \"DeviceName\": \"/dev/xvda\",
      \"Ebs\": { \"VolumeSize\": 20, \"VolumeType\": \"gp3\", \"Encrypted\": true, \"DeleteOnTermination\": true }
    }],
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{ \"Key\": \"Name\", \"Value\": \"lab-mixed\" }]
    }]
  }"
```

## 3. ASG with Mixed Instances Policy

```bash
SUBNET_LIST=$(echo "${SUBNETS[@]}" | tr ' ' ',')

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name lab-mixed-asg \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "$SUBNET_LIST" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB --health-check-grace-period 120 \
  --default-cooldown 60 \
  --capacity-rebalance \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": { "LaunchTemplateName": "lab-mixed-lt", "Version": "$Latest" },
      "Overrides": [
        { "InstanceType": "t4g.small" },
        { "InstanceType": "t4g.medium" },
        { "InstanceType": "m6g.medium" },
        { "InstanceType": "m6g.large" }
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 1,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "price-capacity-optimized",
      "SpotInstancePools": 4
    }
  }' \
  --tags ResourceId=lab-mixed-asg,ResourceType=auto-scaling-group,Key=Environment,Value=lab,PropagateAtLaunch=true
```

## 4. Target Tracking — CPU

```bash
aws autoscaling put-scaling-policy --auto-scaling-group-name lab-mixed-asg \
  --policy-name cpu-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 50.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

## 5. Target Tracking — ALB RPS

```bash
LB_FULL=$(echo $ALB_ARN | awk -F'/' '{print "app/"$3"/"$4}')
TG_FULL=$(echo $TG_ARN | awk -F'/' '{print "targetgroup/"$2"/"$3}')

aws autoscaling put-scaling-policy --auto-scaling-group-name lab-mixed-asg \
  --policy-name rps-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration "{
    \"TargetValue\": 50.0,
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ALBRequestCountPerTarget\",
      \"ResourceLabel\": \"$LB_FULL/$TG_FULL\"
    }
  }"
```

## 6. Predictive Scaling

```bash
aws autoscaling put-scaling-policy --auto-scaling-group-name lab-mixed-asg \
  --policy-name predictive-cpu \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 50.0,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300,
    "MaxCapacityBreachBehavior": "HonorMaxCapacity"
  }'
```

> Predictive 는 14일 history 필요 — 신규 ASG 면 forecast 안 됨. 처음엔 reactive 만.

## 7. Scheduled Action

```bash
aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name lab-mixed-asg \
  --scheduled-action-name peak-morning \
  --recurrence "0 9 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 5 --max-size 15 --desired-capacity 5

aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name lab-mixed-asg \
  --scheduled-action-name rest-night \
  --recurrence "0 21 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 2 --max-size 10 --desired-capacity 2
```

## 8. Warm Pool

```bash
aws autoscaling put-warm-pool --auto-scaling-group-name lab-mixed-asg \
  --min-size 2 \
  --max-group-prepared-capacity 5 \
  --pool-state Stopped \
  --instance-reuse-policy ReuseOnScaleIn=true
```

## 9. Lifecycle Hook

```bash
# SQS
aws sqs create-queue --queue-name lab-asg-hook
HOOK_QUEUE=$(aws sqs get-queue-url --queue-name lab-asg-hook --query QueueUrl --output text)
HOOK_QUEUE_ARN=$(aws sqs get-queue-attributes --queue-url $HOOK_QUEUE --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

# Hook role
aws iam create-role --role-name asg-hook-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"autoscaling.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name asg-hook-role --policy-name p --policy-document "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [{ \"Effect\": \"Allow\", \"Action\": [\"sqs:SendMessage\"], \"Resource\": \"$HOOK_QUEUE_ARN\" }]
}"

ACCT=$(aws sts get-caller-identity --query Account --output text)
aws autoscaling put-lifecycle-hook --auto-scaling-group-name lab-mixed-asg \
  --lifecycle-hook-name launch-hook \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --notification-target-arn $HOOK_QUEUE_ARN \
  --role-arn arn:aws:iam::$ACCT:role/asg-hook-role \
  --heartbeat-timeout 300 \
  --default-result CONTINUE
```

## 10. 부하 시뮬

```bash
DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN --query 'LoadBalancers[0].DNSName' --output text)
sleep 120  # ASG instance ready

# CPU 부하 (instance 안)
aws ssm send-command --instance-ids $(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-mixed-asg --query 'AutoScalingGroups[0].Instances[].InstanceId' --output text | tr '\t' ' ') \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["stress -c 4 -t 600 &"]'

# RPS 부하
ab -n 100000 -c 100 http://$DNS/

# 관찰
watch -n 10 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-mixed-asg --query 'AutoScalingGroups[0].{desired:DesiredCapacity,running:Instances[?LifecycleState==\`InService\`]|length(@)}'"
```

## 11. Scaling Activity 확인

```bash
aws autoscaling describe-scaling-activities --auto-scaling-group-name lab-mixed-asg \
  --max-items 10 \
  --query 'Activities[].{Time:StartTime,Status:StatusCode,Description:Description}'
```

## 12. Predictive Forecast (14일 후)

```bash
aws autoscaling get-predictive-scaling-forecast \
  --auto-scaling-group-name lab-mixed-asg \
  --policy-name predictive-cpu \
  --start-time 2026-05-29T00:00:00Z --end-time 2026-05-30T00:00:00Z
```

## 13. Spot interruption 시뮬

```bash
# 의도적으로 spot termination signal
INST=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-mixed-asg \
  --query 'AutoScalingGroups[0].Instances[?LifecycleState==`InService`]|[0].InstanceId' --output text)

# 시뮬은 어렵지만 capacity-rebalance 시 자동 교체 관찰
aws ec2 describe-instances --instance-ids $INST \
  --query 'Reservations[0].Instances[0].{Lifecycle:InstanceLifecycle}'
```

## 14. Clean up

```bash
aws autoscaling delete-warm-pool --auto-scaling-group-name lab-mixed-asg --force-delete
aws autoscaling delete-lifecycle-hook --auto-scaling-group-name lab-mixed-asg --lifecycle-hook-name launch-hook
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-mixed-asg --force-delete
aws ec2 delete-launch-template --launch-template-name lab-mixed-lt

aws elbv2 delete-listener --listener-arn $(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 delete-target-group --target-group-arn $TG_ARN
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

aws ec2 delete-security-group --group-id $SG_INST
aws ec2 delete-security-group --group-id $SG_ALB

aws sqs delete-queue --queue-url $HOOK_QUEUE
aws iam delete-role-policy --role-name asg-hook-role --policy-name p
aws iam delete-role --role-name asg-hook-role
```

## 15. 학습 포인트

- Mixed Instances Policy 의 4 instance type pool
- `price-capacity-optimized` Spot allocation
- Capacity Rebalance — Spot 회수 전 교체
- Target Tracking 2개 (CPU + RPS) 결합
- Predictive (14일 history 필요)
- Scheduled (peak hour)
- Warm Pool (stopped 빠른 startup)
- Lifecycle Hook + SQS

## 16. 다음

- ▶ [[02-application-auto-scaling-ecs]]
- ▶ [[../03-operations/01-lifecycle-hooks-and-warm-pool]]

## 17. 참고

- [[../auto-scaling]]
- [[../01-fundamentals/01-ec2-asg-deep]]
- [[../02-policies/01-target-tracking]]
