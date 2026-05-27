---
title: "VPC 실습 1 — 3-tier VPC 직접 (Public/Private/Data + NAT + Endpoint)"
kind: practice
status: current
created_at: 2026-05-22T21:42:00+09:00
tags: [aws, vpc, practice, 3-tier, nat-gateway, vpc-endpoint]
home_hub: vpc
---

# 실습 1 — 3-tier VPC

> 목표: VPC + 3 AZ × 3 tier (Public/Private/Data) + NAT GW (HA) + Gateway Endpoint (S3/DDB) + Interface Endpoint (SSM). 60분.

## 1. VPC 생성

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --instance-tenancy default \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-3tier},{Key=Environment,Value=lab}]' \
  --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support '{"Value":true}'

echo "VPC_ID=$VPC_ID"
```

## 2. Subnets (3 AZ × 3 tier = 9 subnet)

```bash
declare -A SUBNETS

# Public (10.0.0.0/24 ~ 10.0.2.0/24)
for i in 0 1 2; do
  AZ_LETTER=$(echo "a b c" | cut -d' ' -f$((i+1)))
  SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.0.$i.0/24 \
    --availability-zone ap-northeast-2$AZ_LETTER \
    --tag-specifications "ResourceType=subnet,Tags=[
      {Key=Name,Value=lab-public-$AZ_LETTER},
      {Key=Tier,Value=public},
      {Key=kubernetes.io/role/elb,Value=1}
    ]" \
    --query 'Subnet.SubnetId' --output text)
  
  # Public subnet — auto-assign public IP
  aws ec2 modify-subnet-attribute --subnet-id $SUBNET --map-public-ip-on-launch
  SUBNETS[public-$AZ_LETTER]=$SUBNET
done

# Private (10.0.10.0/24 ~ 10.0.12.0/24)
for i in 10 11 12; do
  IDX=$((i - 10))
  AZ_LETTER=$(echo "a b c" | cut -d' ' -f$((IDX+1)))
  SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.0.$i.0/24 \
    --availability-zone ap-northeast-2$AZ_LETTER \
    --tag-specifications "ResourceType=subnet,Tags=[
      {Key=Name,Value=lab-private-$AZ_LETTER},
      {Key=Tier,Value=private},
      {Key=kubernetes.io/role/internal-elb,Value=1}
    ]" \
    --query 'Subnet.SubnetId' --output text)
  SUBNETS[private-$AZ_LETTER]=$SUBNET
done

# Data (10.0.20.0/24 ~ 10.0.22.0/24, isolated)
for i in 20 21 22; do
  IDX=$((i - 20))
  AZ_LETTER=$(echo "a b c" | cut -d' ' -f$((IDX+1)))
  SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.0.$i.0/24 \
    --availability-zone ap-northeast-2$AZ_LETTER \
    --tag-specifications "ResourceType=subnet,Tags=[
      {Key=Name,Value=lab-data-$AZ_LETTER},
      {Key=Tier,Value=data}
    ]" \
    --query 'Subnet.SubnetId' --output text)
  SUBNETS[data-$AZ_LETTER]=$SUBNET
done

# 출력
for KEY in "${!SUBNETS[@]}"; do
  echo "$KEY = ${SUBNETS[$KEY]}"
done
```

## 3. Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

## 4. Public Route Table

```bash
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-public-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

# IGW route
aws ec2 create-route --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Public subnets 에 attach
for AZ in a b c; do
  aws ec2 associate-route-table --route-table-id $PUB_RT \
    --subnet-id ${SUBNETS[public-$AZ]}
done
```

## 5. NAT Gateways (per AZ for HA)

```bash
declare -A NAT_GWS

for AZ in a b c; do
  EIP_ALLOC=$(aws ec2 allocate-address --domain vpc \
    --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=lab-nat-eip-$AZ}]" \
    --query AllocationId --output text)
  
  NAT_ID=$(aws ec2 create-nat-gateway \
    --subnet-id ${SUBNETS[public-$AZ]} \
    --allocation-id $EIP_ALLOC \
    --connectivity-type public \
    --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=lab-nat-$AZ}]" \
    --query NatGateway.NatGatewayId --output text)
  
  NAT_GWS[$AZ]=$NAT_ID
done

# Wait
for AZ in a b c; do
  aws ec2 wait nat-gateway-available --nat-gateway-ids ${NAT_GWS[$AZ]}
done
```

## 6. Private Route Tables (per AZ)

```bash
declare -A PRIV_RTS

for AZ in a b c; do
  RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=lab-private-rt-$AZ}]" \
    --query 'RouteTable.RouteTableId' --output text)
  
  # NAT GW route (per AZ)
  aws ec2 create-route --route-table-id $RT \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id ${NAT_GWS[$AZ]}
  
  # Attach
  aws ec2 associate-route-table --route-table-id $RT \
    --subnet-id ${SUBNETS[private-$AZ]}
  
  PRIV_RTS[$AZ]=$RT
done
```

## 7. Data Route Table (isolated, no internet)

```bash
DATA_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-data-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

# No internet route — local 만

for AZ in a b c; do
  aws ec2 associate-route-table --route-table-id $DATA_RT \
    --subnet-id ${SUBNETS[data-$AZ]}
done
```

## 8. Gateway Endpoints (S3 / DynamoDB, 무료)

```bash
# S3
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-northeast-2.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids ${PRIV_RTS[a]} ${PRIV_RTS[b]} ${PRIV_RTS[c]} $DATA_RT

# DynamoDB
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-northeast-2.dynamodb \
  --vpc-endpoint-type Gateway \
  --route-table-ids ${PRIV_RTS[a]} ${PRIV_RTS[b]} ${PRIV_RTS[c]} $DATA_RT
```

## 9. Interface Endpoints (SSM 등)

```bash
# Endpoint SG (HTTPS from clients)
SG_ENDPOINT=$(aws ec2 create-security-group --vpc-id $VPC_ID \
  --group-name lab-endpoint --description "Endpoint SG" \
  --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ENDPOINT \
  --protocol tcp --port 443 --cidr 10.0.0.0/16

# Critical Interface Endpoints
for svc in ssm ssmmessages ec2messages logs; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
    --service-name com.amazonaws.ap-northeast-2.$svc \
    --vpc-endpoint-type Interface \
    --subnet-ids ${SUBNETS[private-a]} ${SUBNETS[private-b]} ${SUBNETS[private-c]} \
    --security-group-ids $SG_ENDPOINT \
    --private-dns-enabled
done
```

## 10. Flow Logs

```bash
# CloudWatch Logs
aws logs create-log-group --log-group-name /aws/vpc/lab-flow
aws logs put-retention-policy --log-group-name /aws/vpc/lab-flow --retention-in-days 30

# IAM role for Flow Logs
aws iam create-role --role-name lab-flow-logs-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"vpc-flow-logs.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name lab-flow-logs-role --policy-name p --policy-document '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents","logs:DescribeLogGroups","logs:DescribeLogStreams"],
    "Resource": "*"
  }]
}'

# Flow Log
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type REJECT \
  --log-destination-type cloud-watch-logs \
  --log-destination "arn:aws:logs:ap-northeast-2:$ACCT:log-group:/aws/vpc/lab-flow" \
  --deliver-logs-permission-arn "arn:aws:iam::$ACCT:role/lab-flow-logs-role"
```

## 11. 검증 — EC2 in private subnet

```bash
SG_TEST=$(aws ec2 create-security-group --vpc-id $VPC_ID \
  --group-name lab-test --description "test" \
  --query GroupId --output text)

AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

INST=$(aws ec2 run-instances --image-id $AMI --instance-type t4g.nano \
  --subnet-id ${SUBNETS[private-a]} \
  --security-group-ids $SG_TEST \
  --iam-instance-profile Name=FirstEc2Profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-private-test}]' \
  --query 'Instances[0].InstanceId' --output text)

sleep 60

# SSM Session Manager 로 접속 (Interface Endpoint 활용)
aws ssm start-session --target $INST

# 안에서
$ curl -s https://s3.ap-northeast-2.amazonaws.com   # Gateway Endpoint OK
$ curl -s https://google.com -m 5                  # NAT GW 통해 OK
$ exit
```

## 12. Reachability Analyzer

```bash
# Internet → Private subnet 의 access 가 차단됐는지 확인 (없어야)
PATH_ID=$(aws ec2 create-network-insights-path \
  --source $IGW_ID \
  --destination $INST \
  --protocol tcp \
  --destination-port 22 \
  --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text)

ANALYSIS_ID=$(aws ec2 start-network-insights-analysis \
  --network-insights-path-id $PATH_ID \
  --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text)

aws ec2 wait network-insights-analysis-status-succeeded --network-insights-analysis-ids $ANALYSIS_ID

aws ec2 describe-network-insights-analyses --network-insights-analysis-ids $ANALYSIS_ID \
  --query 'NetworkInsightsAnalyses[0].NetworkPathFound'
# false (의도된 차단)
```

## 13. Flow Logs 확인

```bash
sleep 60
aws logs tail /aws/vpc/lab-flow --since 5m

# Logs Insights query
aws logs start-query \
  --log-group-name /aws/vpc/lab-flow \
  --start-time $(date -u -v -5M +%s) \
  --end-time $(date -u +%s) \
  --query-string 'fields @timestamp, srcaddr, dstaddr, dstport, action | filter action = "REJECT" | limit 20'
```

## 14. Cost Tag

```bash
for KEY in "${!SUBNETS[@]}"; do
  aws ec2 create-tags --resources ${SUBNETS[$KEY]} \
    --tags Key=CostCenter,Value=engineering Key=Service,Value=bkurs
done

aws ec2 create-tags --resources $VPC_ID $IGW_ID \
  --tags Key=CostCenter,Value=engineering Key=Service,Value=bkurs
```

## 15. Clean up

```bash
aws ec2 terminate-instances --instance-ids $INST
aws ec2 wait instance-terminated --instance-ids $INST

# Endpoints
EP_IDS=$(aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID \
  --query 'VpcEndpoints[].VpcEndpointId' --output text)
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $EP_IDS

# Flow Logs
FL_ID=$(aws ec2 describe-flow-logs --filter Name=resource-id,Values=$VPC_ID \
  --query 'FlowLogs[0].FlowLogId' --output text)
aws ec2 delete-flow-logs --flow-log-ids $FL_ID

# NAT GWs
for AZ in a b c; do
  aws ec2 delete-nat-gateway --nat-gateway-id ${NAT_GWS[$AZ]}
done
sleep 60

# EIPs
for ALLOC in $(aws ec2 describe-addresses --filter Name=tag:Name,Values=lab-nat-eip-* --query 'Addresses[].AllocationId' --output text); do
  aws ec2 release-address --allocation-id $ALLOC
done

# Route Tables (main 제외)
for RT in $PUB_RT $DATA_RT ${PRIV_RTS[a]} ${PRIV_RTS[b]} ${PRIV_RTS[c]}; do
  for ASSOC in $(aws ec2 describe-route-tables --route-table-ids $RT --query 'RouteTables[0].Associations[?Main!=`true`].RouteTableAssociationId' --output text); do
    aws ec2 disassociate-route-table --association-id $ASSOC
  done
  aws ec2 delete-route-table --route-table-id $RT
done

# IGW
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# Subnets
for KEY in "${!SUBNETS[@]}"; do
  aws ec2 delete-subnet --subnet-id ${SUBNETS[$KEY]}
done

# SG
aws ec2 delete-security-group --group-id $SG_TEST
aws ec2 delete-security-group --group-id $SG_ENDPOINT

# Flow Log Role
aws iam delete-role-policy --role-name lab-flow-logs-role --policy-name p
aws iam delete-role --role-name lab-flow-logs-role
aws logs delete-log-group --log-group-name /aws/vpc/lab-flow

# VPC
aws ec2 delete-vpc --vpc-id $VPC_ID
```

## 16. 학습 포인트

- VPC + 9 subnet 의 표준 3-tier
- Per-AZ NAT GW (HA)
- Public RT (IGW) / Private RT per-AZ (NAT) / Data RT (isolated)
- Gateway Endpoint (S3 / DDB) 무료
- Interface Endpoint (SSM / Logs) per-AZ
- Flow Logs (REJECT only, 비용 절감)
- Reachability Analyzer 의 검증 자동화
- EKS subnet tag (kubernetes.io/role/elb / internal-elb)

## 17. 다음

- ▶ [[02-transit-gateway-hub]]
- ▶ [[../../alb/alb]]

## 18. 참고

- [[../vpc]]
- [[../01-fundamentals/01-vpc-and-subnets]]
- [[../01-fundamentals/02-route-tables-and-gateways]]
- [[../02-networking/01-vpc-endpoints-privatelink]]
