---
title: "ECS 실습 3 — Blue-Green deployment with CodeDeploy"
kind: practice
status: current
created_at: 2026-05-22T14:24:00+09:00
tags: [aws, ecs, practice, blue-green, codedeploy]
home_hub: ecs
---

# 실습 3 — Blue-Green deployment with CodeDeploy

> 실습 1 의 Fargate service 위에 CodeDeploy Blue-Green 구성. 60분.

## 1. CodeDeploy IAM role

```bash
aws iam create-role --role-name ecsCodeDeployRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"codedeploy.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy --role-name ecsCodeDeployRole \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS
```

## 2. 2 Target Group + 2 Listener

```bash
TG_B=$(aws elbv2 create-target-group --name echo-blue  --protocol HTTP --port 8080 --vpc-id $VPC --target-type ip --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)
TG_G=$(aws elbv2 create-target-group --name echo-green --protocol HTTP --port 8080 --vpc-id $VPC --target-type ip --health-check-path /health --query 'TargetGroups[0].TargetGroupArn' --output text)

# Prod listener 80 → Blue
LIS_PROD=$(aws elbv2 create-listener --load-balancer-arn $ALB --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TG_B --query 'Listeners[0].ListenerArn' --output text)
# Test listener 8080 → Green
LIS_TEST=$(aws elbv2 create-listener --load-balancer-arn $ALB --protocol HTTP --port 8080 --default-actions Type=forward,TargetGroupArn=$TG_G --query 'Listeners[0].ListenerArn' --output text)

# Test listener SG 열기
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 8080 --cidr 0.0.0.0/0
```

## 3. ECS Service — deploymentController CODE_DEPLOY

```bash
aws ecs create-service --cluster demo --service-name echo-bg \
  --task-definition echo-app --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_LIST],securityGroups=[$SG_TASK],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG_B,containerName=app,containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --deployment-controller "type=CODE_DEPLOY"
```

## 4. CodeDeploy Application + Deployment Group

```bash
aws deploy create-application --application-name echo-bg --compute-platform ECS

aws deploy create-deployment-group \
  --application-name echo-bg \
  --deployment-group-name dg1 \
  --service-role-arn arn:aws:iam::$ACCT:role/ecsCodeDeployRole \
  --deployment-style "deploymentType=BLUE_GREEN,deploymentOption=WITH_TRAFFIC_CONTROL" \
  --blue-green-deployment-configuration "{
    \"terminateBlueInstancesOnDeploymentSuccess\": { \"action\": \"TERMINATE\", \"terminationWaitTimeInMinutes\": 5 },
    \"deploymentReadyOption\": { \"actionOnTimeout\": \"CONTINUE_DEPLOYMENT\" }
  }" \
  --auto-rollback-configuration "enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM" \
  --ecs-services "serviceName=echo-bg,clusterName=demo" \
  --load-balancer-info "{
    \"targetGroupPairInfoList\": [{
      \"targetGroups\": [{\"name\":\"echo-blue\"},{\"name\":\"echo-green\"}],
      \"prodTrafficRoute\": {\"listenerArns\": [\"$LIS_PROD\"]},
      \"testTrafficRoute\": {\"listenerArns\": [\"$LIS_TEST\"]}
    }]
  }" \
  --deployment-config-name CodeDeployDefault.ECSLinear10PercentEvery1Minutes
```

## 5. appspec.yaml

```bash
cat > appspec.yaml <<EOF
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: arn:aws:ecs:$REGION:$ACCT:task-definition/echo-app:2
        LoadBalancerInfo:
          ContainerName: app
          ContainerPort: 8080
EOF
aws s3 mb s3://bkurs-codedeploy-$ACCT 2>/dev/null || true
aws s3 cp appspec.yaml s3://bkurs-codedeploy-$ACCT/appspec.yaml
```

## 6. v2 image + 새 task definition

```bash
# nginx 페이지 변경
sed -i '' 's/hello from ecs/HELLO FROM ECS v2/' default.conf
docker buildx build --platform linux/arm64 -t $ACCT.dkr.ecr.$REGION.amazonaws.com/echo-app:v2 --push .

# new revision
sed 's|echo-app:v1|echo-app:v2|' taskdef.json > taskdef-v2.json
aws ecs register-task-definition --cli-input-json file://taskdef-v2.json
```

## 7. Deployment 시작

```bash
aws deploy create-deployment \
  --application-name echo-bg \
  --deployment-group-name dg1 \
  --revision "revisionType=S3,s3Location={bucket=bkurs-codedeploy-$ACCT,key=appspec.yaml,bundleType=YAML}"
```

→ 새 task definition + green target 시작.

## 8. 흐름 관찰

```bash
DEP_ID=$(aws deploy list-deployments --application-name echo-bg --deployment-group-name dg1 --query 'deployments[0]' --output text)
watch -n 5 "aws deploy get-deployment --deployment-id $DEP_ID --query 'deploymentInfo.{status:status,traffic:rollbackInfo,deployStatus:deploymentOverview}'"
```

## 9. test traffic 확인

```bash
# Green 만 미리 테스트
curl http://$DNS:8080/
# > HELLO FROM ECS v2

# Prod 는 아직 Blue
curl http://$DNS/
# > hello from ecs
```

## 10. Traffic 점진 shift

CodeDeployDefault.ECSLinear10PercentEvery1Minutes — 10% / 1분씩 shift. 10분 후 100%.

```bash
sleep 600
curl http://$DNS/
# > HELLO FROM ECS v2
```

## 11. Rollback 실습

CloudWatch alarm 가 trigger 되면 자동 rollback:

```bash
aws cloudwatch put-metric-alarm --alarm-name echo-5xx \
  --metric-name HTTPCode_Target_5XX_Count --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanThreshold

# Deployment Group 에 alarm 연결
aws deploy update-deployment-group --application-name echo-bg --current-deployment-group-name dg1 \
  --alarm-configuration "enabled=true,alarms=[{name=echo-5xx}]"
```

## 12. Clean up

```bash
aws deploy delete-deployment-group --application-name echo-bg --deployment-group-name dg1
aws deploy delete-application --application-name echo-bg
aws ecs delete-service --cluster demo --service echo-bg --force
aws elbv2 delete-listener --listener-arn $LIS_PROD
aws elbv2 delete-listener --listener-arn $LIS_TEST
aws elbv2 delete-target-group --target-group-arn $TG_B
aws elbv2 delete-target-group --target-group-arn $TG_G
aws s3 rb s3://bkurs-codedeploy-$ACCT --force
```

## 13. 학습 포인트

- prod / test listener 분리 의미
- Linear10PercentEvery1Minutes — 10% / 1분
- terminationWaitTimeInMinutes=5 → rollback 윈도우
- alarm-rollback 자동 trigger
- appspec.yaml 의 task definition 명시

## 14. 다음

- ▶ [[04-spring-boot-end-to-end]]
- ▶ [[../03-service/02-deployment-rolling-and-bg]]

## 15. 참고

- [[../ecs]]
- [[../03-service/02-deployment-rolling-and-bg]]
- [[../04-operations/02-container-insights]]
