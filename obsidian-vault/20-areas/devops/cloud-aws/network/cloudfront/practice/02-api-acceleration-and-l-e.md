---
title: "CloudFront 실습 2 — API Acceleration + Lambda@Edge + Origin Group Failover"
kind: practice
status: current
created_at: 2026-05-23T11:12:00+09:00
tags: [aws, cloudfront, practice, api, lambda-edge, origin-group, failover]
home_hub: cloudfront
---

# 실습 2 — API Acceleration + Lambda@Edge + Origin Group

> 목표: ALB Custom Origin + No cache + Lambda@Edge (auth header inject) + Origin Group failover + Cache Behavior split. 60분.

## 1. Primary + DR ALB

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[?MapPublicIpOnLaunch==`true`]|[0:3].SubnetId' --output text))

SG_ALB=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-cf-alb \
  --description "ALB for CF" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ALB --protocol tcp --port 80 --cidr 0.0.0.0/0

# Primary
ALB_ARN=$(aws elbv2 create-load-balancer --name lab-cf-alb \
  --type application --scheme internet-facing \
  --subnets ${SUBNETS[@]} --security-groups $SG_ALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN --query 'LoadBalancers[0].DNSName' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions 'Type=fixed-response,FixedResponseConfig={ContentType=application/json,MessageBody={"backend":"primary"},StatusCode=200}'

# DR
ALB_DR=$(aws elbv2 create-load-balancer --name lab-cf-alb-dr \
  --type application --scheme internet-facing \
  --subnets ${SUBNETS[@]} --security-groups $SG_ALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
ALB_DR_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_DR --query 'LoadBalancers[0].DNSName' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB_DR --protocol HTTP --port 80 \
  --default-actions 'Type=fixed-response,FixedResponseConfig={ContentType=application/json,MessageBody={"backend":"dr"},StatusCode=200}'
```

## 2. Lambda@Edge (us-east-1 강제)

```bash
aws iam create-role --role-name lab-le-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": ["lambda.amazonaws.com","edgelambda.amazonaws.com"] },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name lab-le-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

cat > le_inject.js <<'EOF'
'use strict';
exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    request.headers['x-cf-secret'] = [{ key: 'X-CF-Secret', value: 'bkurs-secret' }];
    request.headers['x-request-id'] = [{ key: 'X-Request-Id', value: Math.random().toString(36).substring(2) }];
    return request;
};
EOF
zip le_inject.zip le_inject.js

aws lambda create-function --function-name lab-le-inject \
  --runtime nodejs20.x --handler le_inject.handler \
  --role arn:aws:iam::$ACCT:role/lab-le-role \
  --zip-file fileb://le_inject.zip --region us-east-1

LE_VERSION=$(aws lambda publish-version --function-name lab-le-inject \
  --region us-east-1 --query 'Version' --output text)
LE_ARN="arn:aws:lambda:us-east-1:$ACCT:function:lab-le-inject:$LE_VERSION"
```

## 3. CloudFront Distribution + Origin Group

```bash
DIST_JSON=$(cat <<EOF
{
  "CallerReference": "lab-api-cf-$(date +%s)",
  "Comment": "API acceleration + failover",
  "Origins": {
    "Quantity": 2,
    "Items": [
      { "Id": "ALB-primary", "DomainName": "$ALB_DNS",
        "CustomOriginConfig": { "HTTPPort": 80, "HTTPSPort": 443, "OriginProtocolPolicy": "http-only", "OriginSslProtocols": {"Quantity": 1, "Items": ["TLSv1.2"]}, "OriginReadTimeout": 30, "OriginKeepaliveTimeout": 5 } },
      { "Id": "ALB-dr", "DomainName": "$ALB_DR_DNS",
        "CustomOriginConfig": { "HTTPPort": 80, "HTTPSPort": 443, "OriginProtocolPolicy": "http-only", "OriginSslProtocols": {"Quantity": 1, "Items": ["TLSv1.2"]} } }
    ]
  },
  "OriginGroups": {
    "Quantity": 1,
    "Items": [{
      "Id": "alb-failover",
      "FailoverCriteria": { "StatusCodes": { "Quantity": 5, "Items": [500, 502, 503, 504, 403] } },
      "Members": { "Quantity": 2, "Items": [{ "OriginId": "ALB-primary" }, { "OriginId": "ALB-dr" }] }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "alb-failover",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": { "Quantity": 7, "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"] },
    "CachedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] },
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
    "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
    "Compress": true,
    "LambdaFunctionAssociations": {
      "Quantity": 1,
      "Items": [{ "LambdaFunctionARN": "$LE_ARN", "EventType": "origin-request" }]
    }
  },
  "Enabled": true, "PriceClass": "PriceClass_200", "HttpVersion": "http2and3"
}
EOF
)

DIST_ID=$(aws cloudfront create-distribution --distribution-config "$DIST_JSON" --query 'Distribution.Id' --output text)
DIST_DOMAIN=$(aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.DomainName' --output text)
echo "Distribution: $DIST_DOMAIN"
```

## 4. 검증 + Failover

```bash
aws cloudfront wait distribution-deployed --id $DIST_ID

# Normal
curl -L https://$DIST_DOMAIN/api/test
# {"backend":"primary"}

# Primary 강제 fail
LP=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 modify-listener --listener-arn $LP \
  --default-actions 'Type=fixed-response,FixedResponseConfig={ContentType=text/plain,MessageBody=ServerError,StatusCode=503}'

sleep 30
curl -L https://$DIST_DOMAIN/api/test
# 503 → DR 로 자동 → {"backend":"dr"}
```

## 5. Clean up

```bash
# Restore + Disable + Delete
aws elbv2 modify-listener --listener-arn $LP --default-actions 'Type=fixed-response,FixedResponseConfig={ContentType=application/json,MessageBody={"backend":"primary"},StatusCode=200}'

ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)
CONFIG=$(aws cloudfront get-distribution-config --id $DIST_ID --query 'DistributionConfig' | jq '.Enabled = false')
aws cloudfront update-distribution --id $DIST_ID --distribution-config "$CONFIG" --if-match $ETAG
aws cloudfront wait distribution-deployed --id $DIST_ID
aws cloudfront delete-distribution --id $DIST_ID --if-match $(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)

# L@E (replica 제거 ~수분 후)
sleep 600
aws lambda delete-function --function-name lab-le-inject --region us-east-1
aws iam detach-role-policy --role-name lab-le-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name lab-le-role

# ALB cleanup
for A in $ALB_ARN $ALB_DR; do
  for L in $(aws elbv2 describe-listeners --load-balancer-arn $A --query 'Listeners[].ListenerArn' --output text); do
    aws elbv2 delete-listener --listener-arn $L
  done
  aws elbv2 delete-load-balancer --load-balancer-arn $A
done

sleep 60
aws ec2 delete-security-group --group-id $SG_ALB
```

## 6. 학습 포인트

- ALB Custom Origin (http-only)
- Origin Group failover on 5xx/403
- CachingDisabled + AllViewer Origin Request Policy (API)
- Lambda@Edge origin-request (auth header inject)
- L@E version pinning (mutable arn 안 됨, version 명시)
- L@E Logs 의 region-scattered
- Distribution disable + delete 2-step

## 7. 다음

- ▶ [[../cloudfront]]
- ▶ [[../../route53/route53]]

## 8. 참고

- [[../cloudfront]]
- [[01-static-site-cdn]]
- [[../02-features/02-functions-and-lambda-edge]]
