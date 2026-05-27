---
title: "EC2 실습 3 — ALB + Launch Template + ASG (rolling refresh)"
kind: practice
status: current
created_at: 2026-05-22T15:40:00+09:00
tags: [aws, ec2, practice, asg, launch-template, alb]
home_hub: ec2
---

# 실습 3 — ALB + Launch Template + ASG

> 목표: 3-instance ASG + ALB + Target Tracking auto scaling + Rolling Refresh. 60분.

## 1. 네트워크 (default VPC)

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC \
  --query 'Subnets[].SubnetId' --output text))
echo "${SUBNETS[@]}"
```

## 2. SG (ALB + Instance)

```bash
SG_ALB=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name asg-alb --description "alb" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0

SG_INST=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name asg-instance --description "instance" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_INST --protocol tcp --port 80 --source-group $SG_ALB
```

## 3. ALB + Target Group

```bash
ALB=$(aws elbv2 create-load-balancer --name asg-alb --type application \
  --subnets ${SUBNETS[@]} --security-groups $SG_ALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG=$(aws elbv2 create-target-group --name asg-tg --protocol HTTP --port 80 \
  --vpc-id $VPC --target-type instance \
  --health-check-path /health --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG
```

## 4. AMI + Launch Template

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

cat > user-data.sh <<'EOF'
#!/bin/bash
dnf install -y nginx
cat > /usr/share/nginx/html/index.html <<EOH
<h1>v1</h1>
<p>$(hostname)</p>
EOH
cat > /etc/nginx/conf.d/health.conf <<'EOC'
server {
  listen 80;
  location /health { return 200 'ok'; add_header Content-Type text/plain; }
}
EOC
systemctl enable --now nginx
EOF

USER_DATA=$(base64 -i user-data.sh)

aws ec2 create-launch-template --launch-template-name asg-lt \
  --version-description v1 \
  --launch-template-data "{
    \"ImageId\": \"$AMI\",
    \"InstanceType\": \"t4g.small\",
    \"IamInstanceProfile\": { \"Name\": \"FirstEc2Profile\" },
    \"SecurityGroupIds\": [\"$SG_INST\"],
    \"UserData\": \"$USER_DATA\",
    \"MetadataOptions\": { \"HttpTokens\": \"required\", \"HttpPutResponseHopLimit\": 2 },
    \"BlockDeviceMappings\": [{
      \"DeviceName\": \"/dev/xvda\",
      \"Ebs\": { \"VolumeSize\": 20, \"VolumeType\": \"gp3\", \"Encrypted\": true, \"DeleteOnTermination\": true }
    }],
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{ \"Key\": \"Name\", \"Value\": \"asg-app\" }]
    }]
  }"
```

## 5. ASG

```bash
SUBNET_LIST=$(echo "${SUBNETS[@]}" | tr ' ' ',')

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name asg-app \
  --launch-template "LaunchTemplateName=asg-lt,Version=\$Latest" \
  --min-size 2 --max-size 6 --desired-capacity 3 \
  --vpc-zone-identifier "$SUBNET_LIST" \
  --target-group-arns $TG \
  --health-check-type ELB --health-check-grace-period 120 \
  --default-cooldown 60 \
  --capacity-rebalance \
  --tags ResourceId=asg-app,ResourceType=auto-scaling-group,Key=Environment,Value=dev,PropagateAtLaunch=true
```

> `capacityRebalance=true` — Spot 사용 시 interruption 전에 미리 교체.

## 6. Target Tracking Auto Scaling

```bash
aws autoscaling put-scaling-policy --auto-scaling-group-name asg-app \
  --policy-name cpu-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 50.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'

aws autoscaling put-scaling-policy --auto-scaling-group-name asg-app \
  --policy-name request-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration "{
    \"TargetValue\": 1000.0,
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ALBRequestCountPerTarget\",
      \"ResourceLabel\": \"$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB --query 'LoadBalancers[0].LoadBalancerArn' --output text | awk -F'/' '{print $2\"/\"$3\"/\"$4}')/$(aws elbv2 describe-target-groups --target-group-arns $TG --query 'TargetGroups[0].TargetGroupArn' --output text | awk -F'/' '{print $2\"/\"$3\"/\"$4}')\"
    }
  }"
```

## 7. 검증

```bash
DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB --query 'LoadBalancers[0].DNSName' --output text)
sleep 180

for i in 1 2 3 4 5; do
  curl -s http://$DNS/
done
# > 다양한 hostname (3 instance 중 random)
```

## 8. Rolling Refresh (zero-downtime)

```bash
# user-data 변경 → v2
sed -i '' 's/v1/v2/' user-data.sh
USER_DATA=$(base64 -i user-data.sh)

# 새 version
aws ec2 create-launch-template-version \
  --launch-template-name asg-lt \
  --source-version 1 \
  --launch-template-data "{ \"UserData\": \"$USER_DATA\" }"

# ASG 가 $Latest 따라가게
aws autoscaling update-auto-scaling-group --auto-scaling-group-name asg-app \
  --launch-template "LaunchTemplateName=asg-lt,Version=\$Latest"

# Instance Refresh 시작
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name asg-app \
  --strategy Rolling \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 60,
    "ScaleInProtectedInstances": "Ignore",
    "StandbyInstances": "Terminate"
  }'

# 진행 확인
watch -n 10 "aws autoscaling describe-instance-refreshes --auto-scaling-group-name asg-app --query 'InstanceRefreshes[0].{Status:Status,Percentage:PercentageComplete}'"
```

## 9. Load test (scale 트리거)

```bash
# ab — Apache Bench
ab -n 100000 -c 100 http://$DNS/

# ASG 의 변화
watch -n 10 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names asg-app --query 'AutoScalingGroups[0].{desired:DesiredCapacity,running:Instances[?LifecycleState==\`InService\`]|length(@)}'"
```

> CPU 50% 또는 1000 RPS 도달 시 scale-out.

## 10. Scheduled Action

```bash
# 매일 09:00 min 5, 21:00 min 2
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name asg-app --scheduled-action-name morning \
  --recurrence "0 9 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 5 --max-size 10 --desired-capacity 5

aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name asg-app --scheduled-action-name night \
  --recurrence "0 21 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 2 --max-size 6 --desired-capacity 2
```

## 11. Spot Fleet (확장)

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name asg-app \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": { "LaunchTemplateName": "asg-lt", "Version": "$Latest" },
      "Overrides": [
        { "InstanceType": "t4g.small" },
        { "InstanceType": "t4g.medium" },
        { "InstanceType": "m7g.medium" }
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 1,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "capacity-optimized"
    }
  }'
```

→ 75% Spot, 1 base OnDemand 보장.

## 12. Clean up

```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name asg-app --force-delete
aws ec2 delete-launch-template --launch-template-name asg-lt
aws elbv2 delete-listener --listener-arn $(aws elbv2 describe-listeners --load-balancer-arn $ALB --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 delete-target-group --target-group-arn $TG
aws elbv2 delete-load-balancer --load-balancer-arn $ALB
aws ec2 delete-security-group --group-id $SG_INST
aws ec2 delete-security-group --group-id $SG_ALB
```

## 13. 학습 포인트

- Launch Template 의 버전 관리
- ASG 의 health-check-type=ELB (target group 기반)
- Target Tracking (CPU + RPS)
- Instance Refresh = Rolling Refresh
- Mixed Instance Policy 의 Spot + OD 조합
- Capacity Rebalance — Spot interruption 전 교체
- Scheduled Action

## 14. 다음

- ▶ [[../auto-scaling/auto-scaling]]
- ▶ [[../../ecs/practice/02-ec2-with-asg]]
- ▶ [[../03-storage/03-snapshots-ami-lifecycle]]

## 15. 참고

- [[../ec2]]
- [[../01-fundamentals/03-pricing-models]]
- [[../04-operations/02-monitoring-cloudwatch]]
