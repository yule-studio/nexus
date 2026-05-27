---
title: "ECS 실습 1 — Fargate 첫 service (HTTP echo → ALB)"
kind: practice
status: current
created_at: 2026-05-22T14:20:00+09:00
tags: [aws, ecs, practice, fargate, alb]
home_hub: ecs
---

# 실습 1 — Fargate 첫 service

> 목표: nginx 1 task → ALB → 브라우저. 30분.

## 사전 준비

- AWS CLI 설정 + region `ap-northeast-2`
- VPC 1개 (default OK) + public subnet 2개 이상
- session-manager-plugin

## 1. ECR repository

```bash
aws ecr create-repository --repository-name echo-app \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE
```

## 2. Image push

```bash
mkdir echo && cd echo
cat > Dockerfile <<'EOF'
FROM nginxinc/nginx-unprivileged:1.27-alpine
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
EOF
cat > default.conf <<'EOF'
server {
  listen 8080;
  location /health { return 200 'ok'; add_header Content-Type text/plain; }
  location / { return 200 'hello from ecs\n'; add_header Content-Type text/plain; }
}
EOF

ACCT=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-2
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ACCT.dkr.ecr.$REGION.amazonaws.com

docker buildx build --platform linux/arm64 -t $ACCT.dkr.ecr.$REGION.amazonaws.com/echo-app:v1 --push .
```

## 3. IAM role (Execution Role)

```bash
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

## 4. Log group

```bash
aws logs create-log-group --log-group-name /ecs/echo-app
aws logs put-retention-policy --log-group-name /ecs/echo-app --retention-in-days 7
```

## 5. Cluster

```bash
aws ecs create-cluster --cluster-name demo \
  --capacity-providers FARGATE FARGATE_SPOT \
  --settings name=containerInsights,value=enabled
```

## 6. Task definition

```bash
cat > taskdef.json <<EOF
{
  "family": "echo-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "runtimePlatform": { "cpuArchitecture": "ARM64", "operatingSystemFamily": "LINUX" },
  "cpu": "256", "memory": "512",
  "executionRoleArn": "arn:aws:iam::$ACCT:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "app",
    "image": "$ACCT.dkr.ecr.$REGION.amazonaws.com/echo-app:v1",
    "essential": true,
    "portMappings": [{"containerPort": 8080, "protocol": "tcp", "name": "http"}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": { "awslogs-group": "/ecs/echo-app", "awslogs-region": "$REGION", "awslogs-stream-prefix": "app" }
    },
    "healthCheck": { "command": ["CMD-SHELL","wget -q -O- http://localhost:8080/health || exit 1"], "interval": 30, "timeout": 5, "retries": 3 }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://taskdef.json
```

## 7. ALB + Target Group

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[].SubnetId' --output text)

SG_ALB=$(aws ec2 create-security-group --vpc-id $VPC --group-name alb-sg --description "alb" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0

SG_TASK=$(aws ec2 create-security-group --vpc-id $VPC --group-name task-sg --description "task" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_TASK --protocol tcp --port 8080 --source-group $SG_ALB

ALB=$(aws elbv2 create-load-balancer --name echo-alb --subnets $SUBNETS --security-groups $SG_ALB --type application --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG=$(aws elbv2 create-target-group --name echo-tg --protocol HTTP --port 8080 --vpc-id $VPC --target-type ip --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TG
```

## 8. Service

```bash
SUBNET_LIST=$(echo $SUBNETS | tr ' ' ',')
aws ecs create-service --cluster demo --service-name echo \
  --task-definition echo-app \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_LIST],securityGroups=[$SG_TASK],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG,containerName=app,containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --enable-execute-command \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true},maximumPercent=200,minimumHealthyPercent=100"
```

> 실습이라 `assignPublicIp=ENABLED` 임. production 은 private subnet + NAT.

## 9. 검증

```bash
DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB --query 'LoadBalancers[0].DNSName' --output text)
sleep 90  # task starting + ALB registration

curl http://$DNS/
# > hello from ecs

aws logs tail /ecs/echo-app --follow --since 5m
```

## 10. ECS Exec 시도

```bash
TASK=$(aws ecs list-tasks --cluster demo --service-name echo --query 'taskArns[0]' --output text)
aws ecs execute-command --cluster demo --task $TASK --container app --command "/bin/sh" --interactive
```

## 11. Clean up

```bash
aws ecs update-service --cluster demo --service echo --desired-count 0
aws ecs delete-service   --cluster demo --service echo --force
aws elbv2 delete-listener        --listener-arn $(aws elbv2 describe-listeners --load-balancer-arn $ALB --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 delete-target-group    --target-group-arn $TG
aws elbv2 delete-load-balancer   --load-balancer-arn $ALB
aws ecs delete-cluster           --cluster demo
aws logs delete-log-group        --log-group-name /ecs/echo-app
aws ecr delete-repository        --repository-name echo-app --force
aws ec2 delete-security-group --group-id $SG_TASK
aws ec2 delete-security-group --group-id $SG_ALB
aws iam detach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole
```

## 12. 학습 포인트

- Fargate task 의 awsvpc network 확인
- ALB target group ip type 자동 등록
- health check grace period 의 역할
- ECS Exec 으로 shell
- CloudWatch Logs 흐름

## 13. 다음

- ▶ [[02-ec2-with-asg]]
- ▶ [[03-blue-green-with-codedeploy]]
- ▶ [[04-spring-boot-end-to-end]]

## 14. 참고

- [[../ecs]]
- [[../03-service/01-service-overview]]
- [[../03-service/05-load-balancer-integration]]
