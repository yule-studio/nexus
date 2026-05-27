---
title: "ALB 02-3 — WAF + Shield + Access Log"
kind: knowledge
status: current
created_at: 2026-05-23T10:06:00+09:00
tags: [aws, alb, features, waf, shield, access-log, security]
home_hub: alb
---

# ALB WAF + Shield + Access Log

> WAF (L7 OWASP / Rate / Bot) + Shield (DDoS) + Access Log (S3 + Athena). Defense in depth + audit.

자세히 → [[../../vpc/03-security/02-network-firewall-and-waf]]

## 1. WAF Web ACL 활성

```bash
# Web ACL
aws wafv2 create-web-acl \
  --name bkurs-alb \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    { "Name":"AWS-CommonRules", "Priority":1,
      "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesCommonRuleSet"}},
      "OverrideAction":{"None":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"common"} },
    { "Name":"KnownBad", "Priority":2,
      "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesKnownBadInputsRuleSet"}},
      "OverrideAction":{"None":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"bad"} },
    { "Name":"BotControl", "Priority":3,
      "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesBotControlRuleSet"}},
      "OverrideAction":{"None":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"bot"} },
    { "Name":"IpReputation", "Priority":4,
      "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesAmazonIpReputationList"}},
      "OverrideAction":{"None":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"ip"} },
    { "Name":"RateLimit", "Priority":5,
      "Statement":{"RateBasedStatement":{"Limit":2000,"AggregateKeyType":"IP"}},
      "Action":{"Block":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"rate"} }
  ]' \
  --visibility-config "SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=bkurs-alb-waf"

# ALB associate
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:ap-northeast-2:ACCT:regional/webacl/bkurs-alb/xxx \
  --resource-arn $ALB_ARN
```

## 2. WAF Sampled Requests (실시간)

```bash
aws wafv2 get-sampled-requests \
  --web-acl-arn arn:aws:wafv2:...:webacl/bkurs-alb/xxx \
  --rule-metric-name common \
  --scope REGIONAL \
  --time-window StartTime=2026-05-23T09:00:00Z,EndTime=2026-05-23T10:00:00Z \
  --max-items 100
```

→ Block / Count 됐던 request 샘플 확인.

## 3. WAF Custom Rule — Geographic

```bash
aws wafv2 update-web-acl --name bkurs-alb \
  --scope REGIONAL --id $ACL_ID --lock-token $TOKEN \
  --rules '[
    {
      "Name": "BlockNonKorea",
      "Priority": 0,
      "Statement": {
        "NotStatement": {
          "Statement": {
            "GeoMatchStatement": { "CountryCodes": ["KR","US","JP"] }
          }
        }
      },
      "Action": { "Block": {} },
      "VisibilityConfig": { ... }
    }
  ]' ...
```

## 4. WAF Captcha / Challenge

```json
"Action": { "Captcha": {} }
// 또는
"Action": { "Challenge": {} }
```

→ Block 대신 검증 (사용자 vs bot).

## 5. WAF Bot Control

```
AWSManagedRulesBotControlRuleSet:
- common: search engine, monitor, social media — 알려진 bot
- targeted: AI scraping, scraping bots, account takeover bots

Cost: $10/M req (additional)
```

활용:
- E-commerce (가격 scraper)
- Travel (price scraper)
- Account abuse 방지

## 6. WAF Logging

```bash
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:...:webacl/bkurs-alb/xxx",
    "LogDestinationConfigs": [
      "arn:aws:firehose:ap-northeast-2:ACCT:deliverystream/aws-waf-logs-bkurs"
    ],
    "RedactedFields": [{"SingleHeader":{"Name":"authorization"}},{"SingleHeader":{"Name":"cookie"}}]
  }'
```

→ Firehose → S3 / Splunk / OpenSearch.

## 7. AWS Shield

### Shield Standard
- 무료 자동
- L3-L4 attacks (SYN flood, UDP flood)
- ALB / CloudFront / Route 53 / GA

### Shield Advanced ($3000/月)
- L7 attacks 대응
- 24/7 SRT (Shield Response Team)
- DDoS Cost Protection (AWS bill 환불)
- Application Layer DDoS Mitigation (auto)

```bash
aws shield subscribe-to-proactive-engagement \
  --emergency-contact-list EmailAddress=ops@bkurs.com,PhoneNumber=+82-10-...,ContactNotes=primary
```

## 8. Access Log

```bash
# S3 bucket (server access logging activate)
aws s3 mb s3://bkurs-alb-logs
aws s3api put-bucket-policy --bucket bkurs-alb-logs --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "logdelivery.elasticloadbalancing.amazonaws.com" },
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::bkurs-alb-logs/AWSLogs/ACCT/*"
  }]
}'

# ALB attributes
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=bkurs-alb-logs \
    Key=access_logs.s3.prefix,Value=bkurs
```

## 9. Access Log Format

```
2026-05-23T10:00:00.123456Z app/bkurs-prod/abc 203.0.113.10:54321 10.0.10.5:8080 ...
type    time    elb    client:port  target:port  request_processing_time  target_processing_time  response_processing_time
elb_status_code  target_status_code  received_bytes  sent_bytes
"request"  "user_agent"  ssl_cipher  ssl_protocol  target_group_arn  "trace_id"  "domain_name"  "chosen_cert_arn"
matched_rule_priority  request_creation_time  "actions_executed"  "redirect_url"  "error_reason"
"target:port_list"  "target_status_code_list"  "classification"  "classification_reason"
```

| 주요 field |
| --- |
| `client:port` | source IP |
| `target:port` | backend |
| `elb_status_code` | ALB 의 status |
| `target_status_code` | backend 의 status |
| `received_bytes` / `sent_bytes` | 양 |
| `request_processing_time` | ALB → target |
| `target_processing_time` | target 처리 |
| `response_processing_time` | target → client |
| `trace_id` | X-Amzn-Trace-Id (X-Ray) |

## 10. Athena query (Access Log)

```sql
CREATE EXTERNAL TABLE alb_logs (
  type string, time string, elb string, client_port string, target_port string,
  request_processing_time double, target_processing_time double, response_processing_time double,
  elb_status_code int, target_status_code string,
  received_bytes bigint, sent_bytes bigint,
  request_verb string, request_url string, request_proto string,
  user_agent string, ssl_cipher string, ssl_protocol string,
  target_group_arn string, trace_id string, domain_name string,
  chosen_cert_arn string, matched_rule_priority string,
  request_creation_time string, actions_executed string, redirect_url string,
  lambda_error_reason string, target_port_list string, target_status_code_list string,
  classification string, classification_reason string
)
PARTITIONED BY (year int, month int, day int)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'serialization.format' = '1',
  'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ...'
)
LOCATION 's3://bkurs-alb-logs/AWSLogs/ACCT/elasticloadbalancing/...';

-- Top 5xx URLs
SELECT request_url, count(*) as count
FROM alb_logs
WHERE year=2026 AND month=5 AND day=23 AND elb_status_code >= 500
GROUP BY request_url
ORDER BY count DESC LIMIT 10;

-- Top slow URLs
SELECT request_url, avg(target_processing_time) as avg_time
FROM alb_logs
WHERE year=2026 AND month=5
GROUP BY request_url
ORDER BY avg_time DESC LIMIT 10;
```

## 11. Connection Log (2024+)

```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=connection_logs.s3.enabled,Value=true Key=connection_logs.s3.bucket,Value=bkurs-alb-logs Key=connection_logs.s3.prefix,Value=connection
```

→ TLS handshake 정보 (client cipher, SNI, etc).

## 12. CloudWatch alarm — WAF

```bash
aws cloudwatch put-metric-alarm --alarm-name waf-block-spike \
  --metric-name BlockedRequests --namespace AWS/WAFV2 \
  --statistic Sum --period 60 --evaluation-periods 3 \
  --threshold 100 --comparison-operator GreaterThanThreshold \
  --dimensions Name=WebACL,Value=bkurs-alb Name=Region,Value=ap-northeast-2 Name=Rule,Value=ALL
```

## 13. WAF + ALB 비용 시뮬

```
WAF Web ACL:    $5/月
4 Managed Rule: $5 × 4 = $20/月
Bot Control:    extra $10/M req
Request:        10M × $0.60/M = $6/月
= ~$41/月

vs ALB cost ($191):
WAF = 22% overhead
```

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| WAF 없음 → OWASP top 10 위험 | AWS Managed Common Rules |
| Rate limit 없음 → DDoS | Rate-based rule |
| WAF Logging 비활성 | enable Firehose |
| Bot Control 비용 인지 못 함 | per-rule pricing |
| Access Log 비활성 → forensic ↓ | enable |
| Shield Advanced 없음 (critical app) | subscribe |

## 15. 권장 baseline — bkurs-be

```
[ ] WAF Web ACL associate
[ ] AWS Managed Rules:
    - Common Rule Set
    - Known Bad Inputs
    - Amazon IP Reputation List
[ ] Rate-based rule (2000 req/5min/IP)
[ ] Geographic restriction (필요 시)
[ ] WAF Logging → Firehose → S3
[ ] Access Log → S3 (Athena query)
[ ] Connection Log
[ ] Shield Standard 자동
[ ] CloudWatch alarm (BlockedRequests / 5xx)
```

## 16. 다음

- ▶ [[../practice/01-alb-multi-target]]
- ▶ [[../practice/02-blue-green-and-waf]]
- ▶ [[../../nlb/nlb]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [WAF 공식](https://aws.amazon.com/waf/) |
| [Shield](https://aws.amazon.com/shield/) |
| [ALB Access Log](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html) |

## 18. 참고

- [[../alb]]
- [[../../vpc/03-security/02-network-firewall-and-waf]]
