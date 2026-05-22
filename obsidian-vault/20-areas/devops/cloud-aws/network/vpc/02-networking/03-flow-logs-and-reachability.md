---
title: "VPC 02-3 — Flow Logs + Network Insights (Reachability / Access Analyzer)"
kind: knowledge
status: current
created_at: 2026-05-22T21:34:00+09:00
tags: [aws, vpc, networking, flow-logs, reachability, network-access-analyzer]
home_hub: vpc
---

# VPC Flow Logs + Network Insights

> Packet metadata audit (Flow Logs) + Reachability Analyzer (특정 path 검증) + Network Access Analyzer (의도하지 않은 access 탐지).

## 1. VPC Flow Logs

```bash
# 활성 — VPC / Subnet / ENI level
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-destination arn:aws:logs:ap-northeast-2:ACCT:log-group:/aws/vpc/bkurs-flow \
  --deliver-logs-permission-arn arn:aws:iam::ACCT:role/flow-logs-role

# 또는 S3 destination
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::bkurs-flow-logs \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status}'
```

| Traffic Type | 의미 |
| --- | --- |
| `ALL` | 모든 traffic |
| `ACCEPT` | allow 된 traffic 만 |
| `REJECT` | block 된 traffic 만 (audit) |

| Destination |
| --- |
| CloudWatch Logs (실시간 query) |
| S3 (장기 archive + Athena query) |
| Kinesis Data Firehose (Splunk / Sumo Logic) |

## 2. Flow Log Format

### Default (V2)
```
2 ACCT eni-xxx 10.0.1.5 10.0.2.10 49152 80 6 12 6700 1716365400 1716365460 ACCEPT OK
                                       ↑    ↑                 ↑    ↑      ↑                                ↑
                                    srcip dstip          protocol packets bytes                        action
```

### Custom (V5)
| Field | 의미 |
| --- | --- |
| `version` | log version |
| `account-id` | |
| `interface-id` | ENI |
| `srcaddr` / `dstaddr` | IP |
| `srcport` / `dstport` | port |
| `protocol` | 6=TCP, 17=UDP, 1=ICMP |
| `packets` / `bytes` | counter |
| `start` / `end` | timestamp |
| `action` | ACCEPT / REJECT |
| `log-status` | OK / NODATA / SKIPDATA |
| `tcp-flags` | SYN/FIN/RST/ACK |
| `pkt-srcaddr` / `pkt-dstaddr` | 실제 packet (NAT 의 internal IP) |
| `traffic-path` | 1 (in-VPC), 2 (IGW), 3 (VGW), ... |
| `flow-direction` | ingress / egress |
| `vpc-id` / `subnet-id` | resource ID |

## 3. CloudWatch Logs Insights query

```
# Top reject 5
fields @timestamp, srcaddr, dstaddr, action
| filter action = "REJECT"
| stats count() by srcaddr, dstaddr
| sort count desc | limit 5

# Specific IP traffic
fields @timestamp, srcaddr, dstaddr, srcport, dstport, action, bytes
| filter dstaddr like /10.0.10.5/
| sort @timestamp desc | limit 100

# 큰 traffic
fields @timestamp, srcaddr, dstaddr, sum(bytes) as total
| stats sum(bytes) as total by srcaddr, dstaddr
| sort total desc | limit 10
```

## 4. Athena query (S3 destination)

```bash
# Athena table 생성
aws athena start-query-execution --query-string "
CREATE EXTERNAL TABLE bkurs_flow_logs (
  version int, account string, interface_id string,
  srcaddr string, dstaddr string,
  srcport int, dstport int,
  protocol bigint, packets bigint, bytes bigint,
  start_time bigint, end_time bigint,
  action string, log_status string
)
PARTITIONED BY (year int, month int, day int)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
LOCATION 's3://bkurs-flow-logs/AWSLogs/ACCT/vpcflowlogs/ap-northeast-2/'
TBLPROPERTIES ('skip.header.line.count'='1')
" --result-configuration "OutputLocation=s3://athena-results/"

# Query
"SELECT srcaddr, dstaddr, sum(bytes) FROM bkurs_flow_logs 
 WHERE year=2026 AND month=5 AND day=22 AND action='REJECT'
 GROUP BY srcaddr, dstaddr ORDER BY sum(bytes) DESC LIMIT 10"
```

## 5. Reachability Analyzer

특정 source → destination path 검증:
```bash
aws ec2 create-network-insights-path \
  --source $ENI_SOURCE \
  --destination $ENI_DESTINATION \
  --protocol tcp \
  --destination-port 8080 \
  --tag-specifications 'ResourceType=network-insights-path,Tags=[{Key=Name,Value=alb-to-task}]'

# Analysis 실행
aws ec2 start-network-insights-analysis \
  --network-insights-path-id $PATH_ID
```

### 결과 예
```json
{
  "Status": "succeeded",
  "NetworkPathFound": true,
  "ForwardPathComponents": [
    { "ComponentArn": "...:network-interface/eni-source", "Component": {...} },
    { "ComponentArn": "...:security-group/sg-alb", "Component": {...}, "OutboundHeader": {...} },
    { "ComponentArn": "...:network-interface/eni-target", "Component": {...} }
  ]
}
```

### Path 못 찾을 때
```
"NetworkPathFound": false,
"Explanations": [{
  "ExplanationCode": "ENI_SG_RULES_MISMATCH",
  "Direction": "egress",
  "SecurityGroup": {...},
  "PortRanges": [...]
}]
```

→ 어디서 차단됐는지 자동 분석. SG / NACL / Route / Endpoint 등.

## 6. Network Access Analyzer

```bash
# Scope 정의 (analyze 할 access path)
aws ec2 create-network-insights-access-scope \
  --match-paths file://scope.json

# scope.json
{
  "MatchPaths": [{
    "Source": { "ResourceStatement": { "ResourceTypes": ["AWS::EC2::Subnet"] } },
    "Destination": { "ResourceStatement": { "Resources": ["arn:aws:ec2:...:subnet/data-*"] } }
  }],
  "ExcludePaths": []
}

# Analysis
aws ec2 start-network-insights-access-scope-analysis \
  --network-insights-access-scope-id $SCOPE_ID
```

활용:
- "Internet → Data subnet 의 path 가 있는가?" (없어야 함)
- "Public subnet → Database 직접 access?"
- "Cross-account access 가 의도된 만큼인가?"

## 7. Network Manager (Global Network)

```bash
aws networkmanager create-global-network \
  --description "bkurs global network"
```

활용:
- Multi-region / multi-account network 통합 view
- Topology visualization
- Event monitoring (TGW failover, attachment 변경 etc)

## 8. Mirror Session (Traffic Mirroring)

```bash
aws ec2 create-traffic-mirror-target \
  --network-interface-id $TARGET_ENI \
  --description "Security tool"

aws ec2 create-traffic-mirror-filter \
  --description "All traffic"

aws ec2 create-traffic-mirror-session \
  --network-interface-id $SOURCE_ENI \
  --traffic-mirror-target-id $TARGET \
  --traffic-mirror-filter-id $FILTER \
  --session-number 1
```

→ Production traffic 을 IDS / IPS / security appliance 에 복사. Read-only inspection.

## 9. Flow Logs 비용

```
CloudWatch Logs:
  $0.50/GB ingestion + $0.03/GB·월 storage

S3:
  storage 만 ($0.025/GB·월)

Filter:
  - ALL: 큰 volume
  - REJECT only: 작음, audit 적합
```

> 큰 cluster = REJECT only + S3 (Athena query) 권장.

## 10. Reachability Analyzer 비용

```
$0.10 / analysis
+ Network Insights Path 자체는 free
```

## 11. EventBridge 통합

```yaml
EventPattern:
  source: [aws.networkmanager]
  detail-type:
    - "Network Manager Topology Change"
    - "Transit Gateway Network Attachment State Change"
Targets:
  - SNS / Lambda
```

## 12. 권장 baseline 체크리스트

```
[ ] VPC Flow Logs 활성 (REJECT + ALL critical)
[ ] CloudWatch Logs 또는 S3 archive
[ ] Athena table for S3 destination
[ ] Reachability Analyzer (Critical path monthly check)
[ ] Network Access Analyzer (security scope monthly)
[ ] Network Manager (multi-region 있을 시)
[ ] Traffic Mirroring (security tool)
[ ] Flow Logs retention 30-90d
[ ] EventBridge → 알림 (TGW state change)
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Flow Logs 비활성 → 사고 분석 ↓ | enable |
| ALL traffic → 비용 폭주 | REJECT only 또는 S3 |
| Reachability Analyzer 안 활용 | path 변경 시 검증 |
| 사고 후 의도 추적 어려움 | Network Access Analyzer baseline |
| Flow Logs retention 무한 | 30-90d |
| pkt-srcaddr 의 NAT 의미 모름 | V5 format 활용 |

## 14. 다음

- ▶ [[../03-security/01-security-group-vs-nacl]]
- ▶ [[../03-security/02-network-firewall-and-waf]]
- ▶ [[../practice/01-3-tier-vpc]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html) |
| [Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/) |
| [Network Access Analyzer](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/) |

## 16. 참고

- [[../vpc]]
- [[../01-fundamentals/02-route-tables-and-gateways]]
- [[../../../compute/ec2/04-operations/03-troubleshooting]]
