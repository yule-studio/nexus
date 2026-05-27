---
title: "ALB 실습 2 — Blue-Green + WAF + Sticky Session"
kind: practice
status: current
created_at: 2026-05-23T10:12:00+09:00
tags: [aws, alb, practice, blue-green, waf, canary, sticky-session]
home_hub: alb
---

# 실습 2 — Blue-Green + WAF + Canary

> 목표: 2 Target Group (Blue/Green) + Weighted Routing + WAF Web ACL + Geographic restriction + Rate limit. 60분.

## 1. Blue / Green Target Groups

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
# 실습 1 의 VPC / SG_ALB / SG_TARGET 재사용

# Blue TG (v1)
TG_BLUE=$(aws elbv2 create-target-group --name lab-blue \
  --protocol HTTP --port 8080 --vpc-id $VPC --target-type ip \
  --health-check-path / \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Green TG (v2)
TG_GREEN=$(aws elbv2 create-target-group --name lab-green \
  --protocol HTTP --port 8080 --vpc-id $VPC --target-type ip \
  --health-check-path / \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

## 2. ECS Service 로 Blue 채우기 (간단 — Lambda 로 대체)

```bash
# Blue Lambda
cat > blue.py <<'EOF'
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "statusDescription": "200 OK",
        "isBase64Encoded": False,
        "headers": {"Content-Type": "application/json"},
        "body": '{"version":"v1","color":"blue"}'
    }
EOF
zip blue.zip blue.py
aws lambda create-function --function-name lab-blue-fn \
  --runtime python3.12 --architectures arm64 \
  --handler blue.lambda_handler \
  --role arn:aws:iam::$ACCT:role/lab-alb-lambda-role \
  --zip-file fileb://blue.zip

# Green Lambda
cat > green.py <<'EOF'
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "statusDescription": "200 OK",
        "isBase64Encoded": False,
        "headers": {"Content-Type": "application/json"},
        "body": '{"version":"v2","color":"green","feature":"new"}'
    }
EOF
zip green.zip green.py
aws lambda create-function --function-name lab-green-fn \
  --runtime python3.12 --architectures arm64 \
  --handler green.lambda_handler \
  --role arn:aws:iam::$ACCT:role/lab-alb-lambda-role \
  --zip-file fileb://green.zip

# Lambda TG (Blue / Green 별도)
TG_BLUE_L=$(aws elbv2 create-target-group --name lab-blue-l \
  --target-type lambda --health-check-enabled \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
TG_GREEN_L=$(aws elbv2 create-target-group --name lab-green-l \
  --target-type lambda --health-check-enabled \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Permissions
for FN in lab-blue-fn lab-green-fn; do
  TG_VAR=$(echo $FN | sed 's/lab-//;s/-fn//')
  aws lambda add-permission --function-name $FN \
    --statement-id alb --action lambda:InvokeFunction \
    --principal elasticloadbalancing.amazonaws.com \
    --source-arn $(eval echo \$TG_${TG_VAR^^}_L)
done

# Register
aws elbv2 register-targets --target-group-arn $TG_BLUE_L \
  --targets Id="arn:aws:lambda:ap-northeast-2:$ACCT:function:lab-blue-fn"
aws elbv2 register-targets --target-group-arn $TG_GREEN_L \
  --targets Id="arn:aws:lambda:ap-northeast-2:$ACCT:function:lab-green-fn"
```

## 3. ALB + Listener — Canary Weighted

```bash
ALB_ARN=$(aws elbv2 create-load-balancer --name lab-alb-bg \
  --type application --scheme internet-facing \
  --subnets ${SUBNETS[@]} --security-groups $SG_ALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

# Listener — Weighted 90/10
LISTENER_ARN=$(aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions "[{
    \"Type\": \"forward\",
    \"ForwardConfig\": {
      \"TargetGroups\": [
        {\"TargetGroupArn\": \"$TG_BLUE_L\",  \"Weight\": 90},
        {\"TargetGroupArn\": \"$TG_GREEN_L\", \"Weight\": 10}
      ],
      \"TargetGroupStickinessConfig\": { \"Enabled\": false }
    }
  }]" \
  --query 'Listeners[0].ListenerArn' --output text)
```

## 4. Canary 검증

```bash
sleep 30

# 50번 요청 → ~10번 green
for i in $(seq 1 50); do
  curl -s http://$ALB_DNS/
done | sort | uniq -c
# ~45 blue / ~5 green
```

## 5. Progressive Shift (Canary → 50/50 → 100%)

```bash
# 50/50
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions "[{
    \"Type\": \"forward\",
    \"ForwardConfig\": {
      \"TargetGroups\": [
        {\"TargetGroupArn\": \"$TG_BLUE_L\",  \"Weight\": 50},
        {\"TargetGroupArn\": \"$TG_GREEN_L\", \"Weight\": 50}
      ]
    }
  }]"

sleep 60

# 100% Green
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions "Type=forward,TargetGroupArn=$TG_GREEN_L"
```

## 6. Header-based testing (특정 user → Green)

```bash
# header `x-canary: true` 가 있으면 Green
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 5 \
  --conditions '[{
    "Field": "http-header",
    "HttpHeaderConfig": {
      "HttpHeaderName": "x-canary",
      "Values": ["true"]
    }
  }]' \
  --actions "Type=forward,TargetGroupArn=$TG_GREEN_L"

# Test
curl http://$ALB_DNS/ -H "x-canary: true"
# always green
```

## 7. Sticky Session (Target Group Stickiness)

```bash
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions "[{
    \"Type\": \"forward\",
    \"ForwardConfig\": {
      \"TargetGroups\": [
        {\"TargetGroupArn\": \"$TG_BLUE_L\",  \"Weight\": 50},
        {\"TargetGroupArn\": \"$TG_GREEN_L\", \"Weight\": 50}
      ],
      \"TargetGroupStickinessConfig\": {
        \"Enabled\": true,
        \"DurationSeconds\": 3600
      }
    }
  }]"
```

→ 같은 client = 같은 TG (3600s). Canary 의 user-level consistency.

## 8. WAF 활성

```bash
# Web ACL
WEBACL_ID=$(aws wafv2 create-web-acl \
  --name lab-alb-waf --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "CommonRules",
      "Priority": 1,
      "Statement": {"ManagedRuleGroupStatement": {"VendorName":"AWS","Name":"AWSManagedRulesCommonRuleSet"}},
      "OverrideAction": {"None": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "common"}
    },
    {
      "Name": "RateLimit",
      "Priority": 2,
      "Statement": {"RateBasedStatement": {"Limit": 100, "AggregateKeyType": "IP"}},
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "rate"}
    },
    {
      "Name": "GeoKR",
      "Priority": 3,
      "Statement": {
        "NotStatement": {
          "Statement": {
            "GeoMatchStatement": {"CountryCodes": ["KR", "US", "JP"]}
          }
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "geo"}
    }
  ]' \
  --visibility-config "SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=lab-alb-waf" \
  --query 'Summary.Id' --output text)

# ALB associate
WEBACL_ARN=$(aws wafv2 list-web-acls --scope REGIONAL \
  --query "WebACLs[?Name=='lab-alb-waf'].ARN" --output text)
aws wafv2 associate-web-acl \
  --web-acl-arn $WEBACL_ARN \
  --resource-arn $ALB_ARN
```

## 9. Rate Limit test

```bash
# 100 req/5min IP limit
for i in $(seq 1 110); do
  curl -s -o /dev/null -w "%{http_code}\n" http://$ALB_DNS/
done | sort | uniq -c
# 100 200 / 10 403 (rate limited)
```

## 10. WAF Logging

```bash
# Firehose 없이 CloudWatch Logs 로 (간단)
LOG_GROUP=/aws/waf/lab-alb
aws logs create-log-group --log-group-name $LOG_GROUP

aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "'$WEBACL_ARN'",
    "LogDestinationConfigs": ["arn:aws:logs:ap-northeast-2:'$ACCT':log-group:'$LOG_GROUP':*"],
    "RedactedFields": [{"SingleHeader": {"Name": "authorization"}}]
  }'

# Log 확인
sleep 60
aws logs tail $LOG_GROUP --since 5m
```

## 11. CloudWatch alarm

```bash
# Blocked spike
aws cloudwatch put-metric-alarm --alarm-name waf-blocked-spike \
  --metric-name BlockedRequests --namespace AWS/WAFV2 \
  --statistic Sum --period 60 --evaluation-periods 3 \
  --threshold 50 --comparison-operator GreaterThanThreshold \
  --dimensions Name=WebACL,Value=lab-alb-waf Name=Region,Value=ap-northeast-2 Name=Rule,Value=ALL

# Target Group 5xx
aws cloudwatch put-metric-alarm --alarm-name alb-tg-5xx \
  --metric-name HTTPCode_Target_5XX_Count --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --evaluation-periods 3 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --dimensions Name=TargetGroup,Value=$(echo $TG_GREEN_L | awk -F: '{print $6}') Name=LoadBalancer,Value=$(echo $ALB_ARN | awk -F: '{print $6}')
```

## 12. Auto Rollback 시뮬 (CloudWatch Alarm + EventBridge)

```python
# Lambda: rollback on alarm
import boto3
elbv2 = boto3.client("elbv2")

def lambda_handler(event, context):
    state = event["detail"]["state"]["value"]
    if state == "ALARM":
        # 100% Blue 로 rollback
        elbv2.modify_listener(
            ListenerArn=os.environ["LISTENER_ARN"],
            DefaultActions=[{
                "Type": "forward",
                "TargetGroupArn": os.environ["TG_BLUE"]
            }]
        )
        print("Rolled back to Blue")
```

## 13. Clean up

```bash
# Disassociate WAF
aws wafv2 disassociate-web-acl --resource-arn $ALB_ARN

# Web ACL
aws wafv2 delete-web-acl --name lab-alb-waf --scope REGIONAL \
  --id $WEBACL_ID --lock-token $(aws wafv2 get-web-acl --name lab-alb-waf --scope REGIONAL --id $WEBACL_ID --query 'LockToken' --output text)

# Listener / ALB / TG
for L in $(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[].ListenerArn' --output text); do
  aws elbv2 delete-listener --listener-arn $L
done
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

for TG in $TG_BLUE $TG_GREEN $TG_BLUE_L $TG_GREEN_L; do
  aws elbv2 delete-target-group --target-group-arn $TG
done

# Lambda
aws lambda delete-function --function-name lab-blue-fn
aws lambda delete-function --function-name lab-green-fn

# Logs
aws logs delete-log-group --log-group-name $LOG_GROUP
```

## 14. 학습 포인트

- Weighted Forward (Canary deployment)
- Progressive Shift (10% → 50% → 100%)
- Header-based testing (`x-canary`)
- Target Group Stickiness (per-user consistency)
- WAF Web ACL associate to ALB
- Rate-based Rule (per-IP)
- Geographic restriction
- WAF Logging to CloudWatch
- Auto Rollback (Alarm + Lambda)

## 15. 다음

- ▶ [[../alb]]
- ▶ [[../../nlb/nlb]]
- ▶ [[../../../compute/ecs/03-service/02-deployment-rolling-and-bg]]

## 16. 참고

- [[../alb]]
- [[01-alb-multi-target]]
- [[../02-features/03-waf-and-shield]]
- [[../../../compute/ecs/03-service/02-deployment-rolling-and-bg]]
