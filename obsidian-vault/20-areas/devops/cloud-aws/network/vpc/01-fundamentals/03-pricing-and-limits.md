---
title: "VPC 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-22T21:26:00+09:00
tags: [aws, vpc, fundamentals, pricing, limits, ipv4]
home_hub: vpc
---

# VPC Pricing + Limits

> IPv4 cost (2024+) + NAT GW + Data Transfer cross-AZ/region. Hard limits.

## 1. 가격 구성

```
1. IPv4 Address (2024-02+)
2. NAT Gateway
3. Data Transfer (cross-AZ, cross-region, internet)
4. VPC Endpoint (Interface)
5. VPN Connection
6. Transit Gateway
7. Direct Connect
```

## 2. IPv4 Address Pricing (2024-02+)

| Type | $ |
| --- | --- |
| Public IPv4 (in-use, attached) | **$0.005/h** ($3.65/월) |
| Elastic IP (idle, detached) | $0.005/h |
| Reserved IPv4 (BYOIP) | 무료 |

### 비용 절감
```
1. IPv6 활용 (free)
2. Private 만 사용 (public IP 없이)
3. Elastic IP 의 idle release
4. NAT GW 의 multiple EIP 불필요
```

## 3. NAT Gateway Pricing

```
Per NAT GW: $0.045/h ($33/월)
+ DT: $0.045/GB
```

### Cost 시뮬레이션
```
3 NAT GW (per AZ HA) × 730h × $0.045 = $98/월 (고정)
+ 100 GB/일 × 30 × $0.045 = $135/월 (DT)
= $233/월

vs Single NAT GW:
1 × $33 + DT = $33 + $135 = $168/월 (AZ 손실 위험)
```

> Production critical = 3 NAT GW. Cost-sensitive dev = 1 NAT GW.

## 4. Data Transfer Pricing

| 방향 | $ / GB |
| --- | --- |
| Same AZ | $0 |
| Cross-AZ (same region) | $0.01 |
| Cross-region | $0.02 (within US, EU), $0.09 (Asia, etc) |
| Internet outbound | $0.114 (first 10TB), tiered |
| Internet inbound | $0 |
| To CloudFront | $0 |
| To VPC Endpoint same region | $0 |
| Through NAT GW | $0.045/GB |

### Cross-AZ Cost 시뮬
```
ECS task → RDS in another AZ:
1 TB/일 × 30 × $0.01 = $300/월
→ 같은 AZ 우선 placement (Availability Zone Rebalancing)
```

## 5. VPC Endpoint Pricing

| Type | $ |
| --- | --- |
| **Gateway Endpoint (S3 / DynamoDB)** | **무료** |
| **Interface Endpoint** | $7/월/AZ + $0.01/GB |
| **PrivateLink** | $7/월/AZ + $0.01/GB |

### Cost 시뮬레이션 — bkurs-be
```
Gateway Endpoint (S3 + DDB): $0 × 2 = $0
Interface Endpoint × 7 services × 3 AZ:
  (SSM, SecretsManager, Logs, ECR-api, ECR-dkr, KMS, EventBridge)
  = 7 × 3 × $7 = $147/월
+ Traffic $0.01/GB

vs NAT GW DT 절감:
  100 GB/일 × 30 × $0.045 = $135/월 (절감)

→ break-even point. 대량 traffic 시 Endpoint 가 싸짐.
```

## 6. VPN Pricing

```
VPN Connection: $0.05/h ($36/월 per connection)
+ DT: $0.05/GB
```

## 7. Transit Gateway Pricing

```
TGW Attachment: $0.05/h ($36/월) per attachment
+ DT: $0.02/GB processed
```

```
10 VPC × $36 = $360/월
+ 100 GB/일 × 30 × $0.02 = $60/월
= $420/월
```

## 8. Direct Connect Pricing

```
DX Port:
- 1 Gbps:  $0.30/h
- 10 Gbps: $2.25/h
- 100 Gbps: $20/h

Hosted Connection (3rd party):
- 50 Mbps: $0.03/h
- 500 Mbps: $0.20/h
...

DT: free (some tiers) or $0.02-0.05/GB
```

## 9. CloudFront 활용 시 비용 절감

```
S3 / EC2 → CloudFront → Internet:
- S3/EC2 → CF: free
- CF → Internet: $0.085/GB (vs S3/EC2 direct $0.114/GB)

→ 큰 internet outbound 시 CloudFront 가 25% 절감
```

## 10. Hard Limits

| 항목 | Limit |
| --- | --- |
| VPC / region / account | 5 (soft, max 100) |
| Subnet / VPC | 200 (soft, max 200) |
| Route Table / VPC | 200 (soft) |
| Routes / Route Table | 50 (soft, max 1000) |
| Internet Gateway / region | 5 (soft) |
| NAT Gateway / AZ | 5 (soft) |
| Elastic IP / region | 5 (soft, max 50) |
| Security Group / region | 2500 (soft) |
| SG Rules / SG | 60 inbound + 60 outbound (soft, max 500+500) |
| NACL / VPC | 200 (soft) |
| NACL Rules / NACL | 20 inbound + 20 outbound (soft, max 400+400) |
| Network Interface / region | 5000 (soft) |
| VPC Peering / VPC | 50 (soft, max 125) |
| VPC Endpoint / VPC | 50 (soft) |
| VPC Endpoint Service | 20 (soft) |
| TGW / region | 5 (soft) |
| TGW Attachment / TGW | 5000 (soft) |

## 11. Service Quotas 증액

```bash
aws service-quotas list-service-quotas --service-code vpc \
  --query 'Quotas[?Adjustable==`true`].[QuotaName,Value,QuotaCode]' --output table

aws service-quotas request-service-quota-increase \
  --service-code vpc --quota-code L-F678F1CE \
  --desired-value 100
```

## 12. Cost Allocation Tag

```bash
aws ec2 create-tags --resources $VPC_ID $SUBNET_ID $NAT_ID \
  --tags Key=Environment,Value=prod \
         Key=CostCenter,Value=engineering \
         Key=Service,Value=bkurs

aws ce update-cost-allocation-tags-status --cost-allocation-tags-status '[
  { "TagKey": "Environment", "Status": "Active" }
]'
```

## 13. Cost monitoring

### CloudWatch metric
```bash
# NAT GW traffic
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-xxx \
  --period 3600 --statistics Sum

# VPC Endpoint
aws cloudwatch get-metric-statistics --namespace AWS/VPCEndpoint \
  --metric-name BytesProcessed \
  --dimensions Name=VPCEndpointId,Value=vpce-xxx
```

### Cost Explorer
```
Filter: Service = "Amazon Virtual Private Cloud"
Group by: Usage Type
→ DataTransfer-AWS-DataTransfer-In/Out
  + ElasticIP:IdleAddress
  + InfrastructureUsage:NatGateway
```

## 14. 비용 절감 전략

```
1. IPv6 활용 (IPv4 cost 절감)
2. Private subnet + VPC Endpoint (NAT 회피)
3. Same-AZ task placement (cross-AZ DT 절감)
4. CloudFront 활용 (internet outbound)
5. NAT GW 의 traffic 분석 (VPC Endpoint 후보 발견)
6. Elastic IP idle release
7. Endpoint 적절히 (모든 service 가 Endpoint 필요 X)
8. PrivateLink vs Internet 비교
9. Transit Gateway vs Peering (5+ VPC 시 TGW 권장)
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Public IPv4 (2024+) cost 인지 못 함 | IPv6 또는 private |
| NAT GW per VPC 1개 만 | per-AZ HA |
| Cross-AZ DT 비용 무시 | same-AZ placement |
| VPC Endpoint 모든 service | 필요한 것만 (cost vs NAT 비교) |
| Elastic IP idle | release |
| CloudFront 활용 안 함 | static / cache |

## 16. 다음

- ▶ [[../02-networking/01-vpc-endpoints-privatelink]]
- ▶ [[../02-networking/02-peering-and-transit-gateway]]
- ▶ [[../03-security/01-security-group-vs-nacl]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [VPC Pricing](https://aws.amazon.com/vpc/pricing/) |
| [VPC Quotas](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html) |
| [IPv4 charge 발표](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/) |

## 18. 참고

- [[../vpc]]
- [[01-vpc-and-subnets]]
- [[02-route-tables-and-gateways]]
