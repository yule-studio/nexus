---
title: "ECS 실습 2 — EC2 launch + Capacity Provider Managed Scaling"
kind: practice
status: current
created_at: 2026-05-22T14:22:00+09:00
tags: [aws, ecs, practice, ec2, asg, capacity-provider]
home_hub: ecs
---

# 실습 2 — EC2 launch + Capacity Provider Managed Scaling

> 목표: ECS-optimized AMI ASG → Capacity Provider → task. 45분.

## 1. ECS-optimized AMI

```bash
REGION=ap-northeast-2
AMI=$(aws ssm get-parameter --name /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended/image_id --query Parameter.Value --output text)
echo $AMI
```

## 2. IAM

```bash
aws iam create-role --role-name ecsInstanceRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ecsInstanceRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
aws iam create-instance-profile --instance-profile-name ecsInstanceProfile
aws iam add-role-to-instance-profile --instance-profile-name ecsInstanceProfile --role-name ecsInstanceRole
```

## 3. Cluster

```bash
aws ecs create-cluster --cluster-name ec2-demo --settings name=containerInsights,value=enabled
```

## 4. Launch Template

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SG=$(aws ec2 create-security-group --group-name ecs-ec2-sg --description "ECS EC2" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 32768-65535 --cidr 10.0.0.0/8

USER_DATA=$(printf '#!/bin/bash\necho ECS_CLUSTER=ec2-demo >> /etc/ecs/ecs.config\n' | base64)

aws ec2 create-launch-template --launch-template-name ecs-lt \
  --launch-template-data "{
    \"ImageId\": \"$AMI\",
    \"InstanceType\": \"t3.small\",
    \"IamInstanceProfile\": { \"Name\": \"ecsInstanceProfile\" },
    \"SecurityGroupIds\": [\"$SG\"],
    \"UserData\": \"$USER_DATA\"
  }"
```

## 5. Auto Scaling Group

```bash
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[].SubnetId' --output text | tr '\t' ',')
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name ecs-asg \
  --launch-template LaunchTemplateName=ecs-lt \
  --min-size 1 --max-size 5 --desired-capacity 1 \
  --vpc-zone-identifier "$SUBNETS" \
  --new-instances-protected-from-scale-in
```

## 6. Capacity Provider

```bash
ASG_ARN=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ecs-asg --query 'AutoScalingGroups[0].AutoScalingGroupARN' --output text)

aws ecs create-capacity-provider \
  --name ec2-cp \
  --auto-scaling-group-provider "{
    \"autoScalingGroupArn\": \"$ASG_ARN\",
    \"managedScaling\": { \"status\": \"ENABLED\", \"targetCapacity\": 100, \"minimumScalingStepSize\": 1, \"maximumScalingStepSize\": 5, \"instanceWarmupPeriod\": 300 },
    \"managedTerminationProtection\": \"ENABLED\"
  }"

aws ecs put-cluster-capacity-providers \
  --cluster ec2-demo \
  --capacity-providers ec2-cp \
  --default-capacity-provider-strategy "capacityProvider=ec2-cp,weight=1,base=1"
```

## 7. Task definition (bridge)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
cat > td.json <<EOF
{
  "family": "ec2-demo",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "executionRoleArn": "arn:aws:iam::$ACCT:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "app",
    "image": "nginxdemos/hello",
    "essential": true,
    "memory": 128,
    "portMappings": [{ "containerPort": 80, "hostPort": 0 }],
    "logConfiguration": { "logDriver": "awslogs", "options": { "awslogs-group": "/ecs/ec2-demo", "awslogs-region": "$REGION", "awslogs-stream-prefix": "app", "awslogs-create-group": "true" } }
  }]
}
EOF
aws ecs register-task-definition --cli-input-json file://td.json
```

## 8. Service

```bash
aws ecs create-service --cluster ec2-demo --service-name web \
  --task-definition ec2-demo --desired-count 3 \
  --capacity-provider-strategy capacityProvider=ec2-cp,weight=1
```

## 9. 검증 — Managed Scaling

```bash
# 3 task → 1 EC2 부족 → ASG scale out 자동
watch -n 5 'aws ecs describe-services --cluster ec2-demo --services web --query "services[0].{desired:desiredCount,running:runningCount,pending:pendingCount}"'

# instance 증가
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ecs-asg \
  --query 'AutoScalingGroups[0].{desired:DesiredCapacity,instances:Instances[].InstanceId}'

# desired 10 → 추가 scale
aws ecs update-service --cluster ec2-demo --service web --desired-count 10
```

## 10. Clean up

```bash
aws ecs update-service --cluster ec2-demo --service web --desired-count 0
aws ecs delete-service --cluster ec2-demo --service web --force
aws ecs put-cluster-capacity-providers --cluster ec2-demo --capacity-providers --default-capacity-provider-strategy
aws ecs delete-capacity-provider --capacity-provider ec2-cp
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ecs-asg --min-size 0 --desired-capacity 0
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name ecs-asg --force-delete
aws ec2 delete-launch-template --launch-template-name ecs-lt
aws ec2 delete-security-group --group-id $SG
aws ecs delete-cluster --cluster ec2-demo
aws logs delete-log-group --log-group-name /ecs/ec2-demo
aws iam remove-role-from-instance-profile --instance-profile-name ecsInstanceProfile --role-name ecsInstanceRole
aws iam delete-instance-profile --instance-profile-name ecsInstanceProfile
aws iam detach-role-policy --role-name ecsInstanceRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
aws iam delete-role --role-name ecsInstanceRole
```

## 11. 학습 포인트

- ECS-optimized AMI + user data 의 `ECS_CLUSTER` env
- ASG + Capacity Provider 의 Managed Scaling 동작
- bridge network mode 의 dynamic port (32768-65535)
- managedTerminationProtection 가 scale-in 시 task 가진 instance 보호

## 12. 다음

- ▶ [[03-blue-green-with-codedeploy]]
- ▶ [[04-spring-boot-end-to-end]]
- ▶ [[../03-service/03-auto-scaling]]

## 13. 참고

- [[../ecs]]
- [[../01-fundamentals/02-launch-type-and-capacity-provider]]
