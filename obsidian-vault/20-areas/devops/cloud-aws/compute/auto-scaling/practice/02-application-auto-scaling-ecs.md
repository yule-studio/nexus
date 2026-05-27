---
title: "Auto Scaling 실습 2 — ECS Service + CPU+RPS+Scheduled + DDB"
kind: practice
status: current
created_at: 2026-05-22T18:24:00+09:00
tags: [aws, auto-scaling, practice, ecs, application-auto-scaling]
home_hub: auto-scaling
---

# 실습 2 — ECS Service + Application Auto Scaling

> 목표: ECS service 의 Auto Scaling 3 policy (CPU / RPS / Scheduled) + DDB Auto Scaling. 45분.

> 기존 ECS cluster (또는 [[../../ecs/practice/01-fargate-first-service]]) 위에서 진행.

## 1. ECS service + ALB 구성

(실습 1 의 fargate cluster + ALB + TG 재사용)

```bash
CLUSTER=demo                         # 실습 1 의 cluster
SERVICE=echo                          # 실습 1 의 service
TG_ARN=$(aws elbv2 describe-target-groups --names echo-tg --query 'TargetGroups[0].TargetGroupArn' --output text)
ALB_ARN=$(aws elbv2 describe-load-balancers --names echo-alb --query 'LoadBalancers[0].LoadBalancerArn' --output text)
```

## 2. Scalable Target 등록

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 10
```

## 3. Target Tracking — CPU 60%

```bash
aws application-autoscaling put-scaling-policy \
  --policy-name ecs-cpu-60 \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

## 4. Target Tracking — Memory 70%

```bash
aws application-autoscaling put-scaling-policy \
  --policy-name ecs-mem-70 \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    }
  }'
```

## 5. Target Tracking — ALB RPS

```bash
LB_FULL=$(echo $ALB_ARN | awk -F'/' '{print "app/"$3"/"$4}')
TG_FULL=$(echo $TG_ARN | awk -F'/' '{print "targetgroup/"$2"/"$3}')

aws application-autoscaling put-scaling-policy \
  --policy-name ecs-rps-100 \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration "{
    \"TargetValue\": 100.0,
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ALBRequestCountPerTarget\",
      \"ResourceLabel\": \"$LB_FULL/$TG_FULL\"
    }
  }"
```

## 6. Scheduled Action

```bash
# 매일 09:00 → min 5
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name morning \
  --schedule "cron(0 9 ? * MON-FRI *)" --timezone "Asia/Seoul" \
  --scalable-target-action "MinCapacity=5,MaxCapacity=20"

aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name evening \
  --schedule "cron(0 21 ? * MON-FRI *)" --timezone "Asia/Seoul" \
  --scalable-target-action "MinCapacity=2,MaxCapacity=10"
```

## 7. DynamoDB Auto Scaling

```bash
aws dynamodb create-table --table-name lab-counter \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5

aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/lab-counter" \
  --scalable-dimension dynamodb:table:WriteCapacityUnits \
  --min-capacity 5 --max-capacity 1000

aws application-autoscaling put-scaling-policy \
  --policy-name ddb-write \
  --service-namespace dynamodb \
  --resource-id "table/lab-counter" \
  --scalable-dimension dynamodb:table:WriteCapacityUnits \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBWriteCapacityUtilization"
    }
  }'

# Read 도
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id "table/lab-counter" \
  --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --min-capacity 5 --max-capacity 1000

aws application-autoscaling put-scaling-policy \
  --policy-name ddb-read \
  --service-namespace dynamodb \
  --resource-id "table/lab-counter" \
  --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
    }
  }'
```

## 8. 부하 시뮬

```bash
DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN --query 'LoadBalancers[0].DNSName' --output text)

# RPS 부하
ab -n 200000 -c 200 http://$DNS/

# 관찰
watch -n 10 "aws ecs describe-services --cluster $CLUSTER --services $SERVICE --query 'services[0].{desired:desiredCount,running:runningCount,pending:pendingCount}'"

# Scaling activity
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --max-items 10 \
  --query 'ScalingActivities[].{Time:StartTime,Status:StatusCode,Description:Description}'
```

## 9. DDB 부하

```bash
for i in $(seq 1 10000); do
  aws dynamodb put-item --table-name lab-counter \
    --item "{\"id\":{\"S\":\"item-$i\"},\"timestamp\":{\"S\":\"$(date -Iseconds)\"}}" &
done
wait

# DDB capacity 자동 증가 확인
watch -n 30 "aws dynamodb describe-table --table-name lab-counter --query 'Table.ProvisionedThroughput.{Read:ReadCapacityUnits,Write:WriteCapacityUnits}'"
```

## 10. List policies / actions

```bash
# ECS 의 scaling policies
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --query 'ScalingPolicies[].[PolicyName,PolicyType]' --output table

# Scheduled actions
aws application-autoscaling describe-scheduled-actions \
  --service-namespace ecs \
  --query 'ScheduledActions[].[ScheduledActionName,Schedule]' --output table

# Activities
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs --max-items 20
```

## 11. Clean up

```bash
# ECS Application Auto Scaling
for p in ecs-cpu-60 ecs-mem-70 ecs-rps-100; do
  aws application-autoscaling delete-scaling-policy \
    --service-namespace ecs --policy-name $p \
    --resource-id service/$CLUSTER/$SERVICE \
    --scalable-dimension ecs:service:DesiredCount
done

for s in morning evening; do
  aws application-autoscaling delete-scheduled-action \
    --service-namespace ecs --scheduled-action-name $s \
    --resource-id service/$CLUSTER/$SERVICE \
    --scalable-dimension ecs:service:DesiredCount
done

aws application-autoscaling deregister-scalable-target \
  --service-namespace ecs \
  --resource-id service/$CLUSTER/$SERVICE \
  --scalable-dimension ecs:service:DesiredCount

# DDB
for p in ddb-read ddb-write; do
  D=$(echo $p | sed 's/ddb-//')
  aws application-autoscaling delete-scaling-policy \
    --service-namespace dynamodb --policy-name $p \
    --resource-id "table/lab-counter" \
    --scalable-dimension "dynamodb:table:${D^}CapacityUnits"
done

for d in Read Write; do
  aws application-autoscaling deregister-scalable-target \
    --service-namespace dynamodb \
    --resource-id "table/lab-counter" \
    --scalable-dimension "dynamodb:table:${d}CapacityUnits"
done

aws dynamodb delete-table --table-name lab-counter
```

## 12. 학습 포인트

- Application Auto Scaling 의 register-scalable-target 패턴
- 3 policy 결합 (CPU + Memory + RPS) — 어느 거든 trigger
- ResourceLabel 형식 (ALB + TG)
- Scheduled 가 min/max 변경 / TT 가 desired 자동
- DDB Provisioned mode 의 Read+Write 자동 scaling
- ECS Auto Scaling vs ECS Capacity Provider 의 layer 차이

## 13. 다음

- ▶ [[../auto-scaling]]
- ▶ [[../../ecs/03-service/03-auto-scaling]]
- ▶ [[../../database/dynamodb/dynamodb]]

## 14. 참고

- [[../auto-scaling]]
- [[01-asg-with-mixed-instances]]
- [[../01-fundamentals/02-application-auto-scaling]]
- [[../02-policies/01-target-tracking]]
- [[../02-policies/03-scheduled]]
