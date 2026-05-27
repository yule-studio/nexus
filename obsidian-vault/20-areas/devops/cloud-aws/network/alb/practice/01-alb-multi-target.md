---
title: "ALB 실습 1 — ALB + Multi-TG (ECS + Lambda + EC2)"
kind: practice
status: current
created_at: 2026-05-23T10:10:00+09:00
tags: [aws, alb, practice, ecs, lambda, ec2, multi-target]
home_hub: alb
---

# 실습 1 — ALB + Multi Target Group

> 목표: ALB + 4 Target Group (ECS IP / Lambda / EC2 / Static) + Path routing + HTTPS + ACM. 60분.

## 1. 기본 인프라 (VPC + SG)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[].SubnetId' --output text))

SG_ALB=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-alb \
  --description "ALB" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 443 --cidr 0.0.0.0/0

SG_TARGET=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-alb-target \
  --description "ALB Targets" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_TARGET \
  --protocol tcp --port 8080 --source-group $SG_ALB
```

## 2. ALB 생성

```bash
ALB_ARN=$(aws elbv2 create-load-balancer --name lab-alb \
  --type application --scheme internet-facing \
  --subnets ${SUBNETS[@]} --security-groups $SG_ALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)
echo "ALB DNS: $ALB_DNS"
```

## 3. 4 Target Groups

```bash
# TG 1: ECS IP target
TG_ECS=$(aws elbv2 create-target-group --name lab-tg-ecs \
  --protocol HTTP --port 8080 \
  --vpc-id $VPC --target-type ip \
  --health-check-path /health --health-check-interval-seconds 15 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# TG 2: Lambda
TG_LAMBDA=$(aws elbv2 create-target-group --name lab-tg-lambda \
  --target-type lambda --health-check-enabled \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# TG 3: EC2 instance
TG_EC2=$(aws elbv2 create-target-group --name lab-tg-ec2 \
  --protocol HTTP --port 80 \
  --vpc-id $VPC --target-type instance \
  --health-check-path / \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Default 404 TG (또는 fixed-response)
TG_DEFAULT=$(aws elbv2 create-target-group --name lab-tg-default \
  --protocol HTTP --port 8080 \
  --vpc-id $VPC --target-type ip \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

## 4. EC2 instance (Target 3)

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y nginx
echo "<h1>From EC2</h1>" > /usr/share/nginx/html/index.html
systemctl enable --now nginx
EOF

INST=$(aws ec2 run-instances --image-id $AMI --instance-type t4g.nano \
  --subnet-id ${SUBNETS[0]} --security-group-ids $SG_TARGET \
  --iam-instance-profile Name=FirstEc2Profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-alb-ec2}]' \
  --query 'Instances[0].InstanceId' --output text)

sleep 60

# Register
aws elbv2 register-targets --target-group-arn $TG_EC2 --targets Id=$INST,Port=80
```

EC2 의 SG 80 inbound:
```bash
aws ec2 authorize-security-group-ingress --group-id $SG_TARGET \
  --protocol tcp --port 80 --source-group $SG_ALB
```

## 5. Lambda function (Target 2)

```bash
cat > lambda.py <<'EOF'
def lambda_handler(event, context):
    path = event.get("path", "/")
    return {
        "statusCode": 200,
        "statusDescription": "200 OK",
        "isBase64Encoded": False,
        "headers": {"Content-Type": "text/html"},
        "body": f"<h1>From Lambda</h1><p>Path: {path}</p>"
    }
EOF
zip lambda.zip lambda.py

aws iam create-role --role-name lab-alb-lambda-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name lab-alb-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws lambda create-function --function-name lab-alb-fn \
  --runtime python3.12 --architectures arm64 \
  --handler lambda.lambda_handler \
  --role arn:aws:iam::$ACCT:role/lab-alb-lambda-role \
  --zip-file fileb://lambda.zip

# ALB permission
aws lambda add-permission --function-name lab-alb-fn \
  --statement-id alb --action lambda:InvokeFunction \
  --principal elasticloadbalancing.amazonaws.com \
  --source-arn $TG_LAMBDA

# Register
aws elbv2 register-targets --target-group-arn $TG_LAMBDA \
  --targets Id="arn:aws:lambda:ap-northeast-2:$ACCT:function:lab-alb-fn"
```

## 6. Default Listener — 404 + redirect HTTP→HTTPS

```bash
# HTTP 80 (redirect)
aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions 'Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# HTTPS 443 (default 404 fixed-response)
LISTENER_ARN=$(aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 443 \
  --default-actions 'Type=fixed-response,FixedResponseConfig={ContentType=text/html,MessageBody=<h1>404 Not Found</h1>,StatusCode=404}' \
  --query 'Listeners[0].ListenerArn' --output text)
# 실습 — HTTPS cert 없이 HTTP 443
# Production = certificates CertificateArn=...
```

## 7. Routing Rules

```bash
# Rule 1: /api/* → ECS TG
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 10 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/api/*"]}}]' \
  --actions "Type=forward,TargetGroupArn=$TG_ECS"

# Rule 2: /lambda/* → Lambda TG
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 20 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/lambda/*"]}}]' \
  --actions "Type=forward,TargetGroupArn=$TG_LAMBDA"

# Rule 3: /ec2/* → EC2 TG
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 30 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/ec2/*"]}}]' \
  --actions "Type=forward,TargetGroupArn=$TG_EC2"

# Rule 4: Health endpoint (ALB level fixed-response)
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 5 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/healthz"]}}]' \
  --actions 'Type=fixed-response,FixedResponseConfig={ContentType=text/plain,MessageBody=OK,StatusCode=200}'
```

## 8. Weighted Routing (Blue-Green / Canary)

```bash
# 90% TG_ECS / 10% TG_LAMBDA (canary)
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 40 \
  --conditions '[{"Field":"path-pattern","PathPatternConfig":{"Values":["/v2/*"]}}]' \
  --actions "[{
    \"Type\": \"forward\",
    \"ForwardConfig\": {
      \"TargetGroups\": [
        {\"TargetGroupArn\": \"$TG_ECS\",    \"Weight\": 90},
        {\"TargetGroupArn\": \"$TG_LAMBDA\", \"Weight\": 10}
      ]
    }
  }]"
```

## 9. 검증

```bash
sleep 60

# Health
curl http://$ALB_DNS/healthz
# OK

# Lambda
curl http://$ALB_DNS/lambda/test
# <h1>From Lambda</h1>

# EC2
curl http://$ALB_DNS/ec2/
# <h1>From EC2</h1>

# Default 404
curl -i http://$ALB_DNS/unknown
# 404 Not Found
```

## 10. CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=$(echo $ALB_ARN | awk -F'/' '{print "app/"$3"/"$4}') \
  --period 60 --statistics Sum
```

## 11. Access Log 활성

```bash
BUCKET=lab-alb-logs-$ACCT-$(date +%s)
aws s3 mb s3://$BUCKET

# Bucket policy
aws s3api put-bucket-policy --bucket $BUCKET --policy "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [{
    \"Effect\": \"Allow\",
    \"Principal\": { \"Service\": \"logdelivery.elasticloadbalancing.amazonaws.com\" },
    \"Action\": \"s3:PutObject\",
    \"Resource\": \"arn:aws:s3:::$BUCKET/AWSLogs/$ACCT/*\"
  }]
}"

aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=$BUCKET \
    Key=access_logs.s3.prefix,Value=lab-alb

# 5분 후 확인
sleep 300
aws s3 ls s3://$BUCKET/lab-alb/AWSLogs/$ACCT/elasticloadbalancing/ --recursive
```

## 12. Clean up

```bash
# Listener (delete first)
for L in $(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[].ListenerArn' --output text); do
  aws elbv2 delete-listener --listener-arn $L
done

# ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# Target Groups
for TG in $TG_ECS $TG_LAMBDA $TG_EC2 $TG_DEFAULT; do
  aws elbv2 delete-target-group --target-group-arn $TG
done

# EC2
aws ec2 terminate-instances --instance-ids $INST

# Lambda
aws lambda delete-function --function-name lab-alb-fn
aws iam detach-role-policy --role-name lab-alb-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name lab-alb-lambda-role

# SG (instance terminate 후)
sleep 60
aws ec2 delete-security-group --group-id $SG_TARGET
aws ec2 delete-security-group --group-id $SG_ALB

# S3
aws s3 rb s3://$BUCKET --force
```

## 13. 학습 포인트

- ALB + Multi target type (IP / Lambda / Instance)
- Path-based routing (Rule priority)
- Weighted Forward (Canary)
- HTTP → HTTPS redirect (default action)
- Fixed-response (404 / 200 health)
- ALB Access Log → S3
- Lambda Target 의 `statusDescription` 필수

## 14. 다음

- ▶ [[02-blue-green-and-waf]]
- ▶ [[../02-features/01-https-and-acm]]
- ▶ [[../02-features/03-waf-and-shield]]

## 15. 참고

- [[../alb]]
- [[../01-fundamentals/01-listener-and-target-group]]
- [[../01-fundamentals/02-routing-rules-and-conditions]]
