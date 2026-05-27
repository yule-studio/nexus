---
title: "VPC 실습 2 — Multi-VPC + Transit Gateway Hub"
kind: practice
status: current
created_at: 2026-05-22T21:44:00+09:00
tags: [aws, vpc, practice, transit-gateway, multi-vpc, hub-spoke]
home_hub: vpc
---

# 실습 2 — Multi-VPC + Transit Gateway

> 목표: 3 VPC (Prod / Dev / Shared) + TGW hub + Route Table isolation. 90분.

## 1. 3 VPC 생성

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

declare -A VPCS

for NAME in prod dev shared; do
  case $NAME in
    prod) CIDR=10.10.0.0/16 ;;
    dev) CIDR=10.20.0.0/16 ;;
    shared) CIDR=10.30.0.0/16 ;;
  esac
  
  VPC_ID=$(aws ec2 create-vpc --cidr-block $CIDR \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc-$NAME}]" \
    --query 'Vpc.VpcId' --output text)
  
  aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames '{"Value":true}'
  aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support '{"Value":true}'
  
  VPCS[$NAME]=$VPC_ID
done

echo "VPCs: ${VPCS[@]}"
```

## 2. 각 VPC 의 Subnet (per AZ)

```bash
declare -A SUBNETS

for NAME in prod dev shared; do
  case $NAME in
    prod) BASE=10.10 ;;
    dev) BASE=10.20 ;;
    shared) BASE=10.30 ;;
  esac
  
  for i in 0 1 2; do
    AZ_LETTER=$(echo "a b c" | cut -d' ' -f$((i+1)))
    SUBNET=$(aws ec2 create-subnet --vpc-id ${VPCS[$NAME]} \
      --cidr-block $BASE.$i.0/24 \
      --availability-zone ap-northeast-2$AZ_LETTER \
      --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=lab-$NAME-$AZ_LETTER}]" \
      --query 'Subnet.SubnetId' --output text)
    SUBNETS[$NAME-$AZ_LETTER]=$SUBNET
  done
done
```

## 3. Transit Gateway 생성

```bash
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "lab hub" \
  --options "AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable,VpnEcmpSupport=enable,DnsSupport=enable" \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=lab-tgw}]' \
  --query 'TransitGateway.TransitGatewayId' --output text)

aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID
echo "TGW_ID=$TGW_ID"
```

> `DefaultRouteTableAssociation=disable` — Custom RT 활용 위함.

## 4. TGW Route Tables (Custom)

```bash
TGW_RT_PROD=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=lab-tgw-rt-prod}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

TGW_RT_DEV=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=lab-tgw-rt-dev}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

TGW_RT_SHARED=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=lab-tgw-rt-shared}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)
```

## 5. VPC Attachments

```bash
declare -A TGW_ATTS

for NAME in prod dev shared; do
  ATT=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id ${VPCS[$NAME]} \
    --subnet-ids ${SUBNETS[$NAME-a]} ${SUBNETS[$NAME-b]} ${SUBNETS[$NAME-c]} \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=lab-tgw-att-$NAME}]" \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)
  TGW_ATTS[$NAME]=$ATT
done

# Wait for available
sleep 30
```

## 6. RT Association + Propagation (Isolation Pattern)

```bash
# Prod attachment → Prod RT (association)
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-attachment-id ${TGW_ATTS[prod]} \
  --transit-gateway-route-table-id $TGW_RT_PROD

# Prod ↔ Shared (propagation)
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-attachment-id ${TGW_ATTS[prod]} \
  --transit-gateway-route-table-id $TGW_RT_SHARED
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-attachment-id ${TGW_ATTS[shared]} \
  --transit-gateway-route-table-id $TGW_RT_PROD

# Dev attachment → Dev RT
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-attachment-id ${TGW_ATTS[dev]} \
  --transit-gateway-route-table-id $TGW_RT_DEV

# Dev ↔ Shared (Prod 와 isolated)
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-attachment-id ${TGW_ATTS[dev]} \
  --transit-gateway-route-table-id $TGW_RT_SHARED
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-attachment-id ${TGW_ATTS[shared]} \
  --transit-gateway-route-table-id $TGW_RT_DEV

# Shared attachment → Shared RT
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-attachment-id ${TGW_ATTS[shared]} \
  --transit-gateway-route-table-id $TGW_RT_SHARED
```

**결과**:
```
Prod   ↔ Shared    OK
Dev    ↔ Shared    OK
Prod   ↔ Dev       차단
```

## 7. VPC Route Tables 에 TGW route 추가

```bash
for NAME in prod dev shared; do
  RT=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=${VPCS[$NAME]} \
    --query 'RouteTables[?Associations[?Main==`true`]].RouteTableId' --output text)
  
  # 다른 VPC 들로 가는 traffic = TGW
  aws ec2 create-route --route-table-id $RT \
    --destination-cidr-block 10.0.0.0/8 \
    --transit-gateway-id $TGW_ID
done
```

## 8. SG (모든 VPC 의 traffic 허용 — 실습)

```bash
declare -A SGS

for NAME in prod dev shared; do
  SG=$(aws ec2 create-security-group --vpc-id ${VPCS[$NAME]} \
    --group-name lab-$NAME-test --description "$NAME" \
    --query GroupId --output text)
  
  # Allow VPC CIDR (10.0.0.0/8 across all)
  aws ec2 authorize-security-group-ingress --group-id $SG \
    --protocol icmp --port -1 --cidr 10.0.0.0/8
  aws ec2 authorize-security-group-ingress --group-id $SG \
    --protocol tcp --port 22 --cidr 10.0.0.0/8
  
  SGS[$NAME]=$SG
done
```

## 9. Test EC2 (각 VPC 에)

```bash
declare -A INSTANCES

AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

for NAME in prod dev shared; do
  INST=$(aws ec2 run-instances --image-id $AMI --instance-type t4g.nano \
    --subnet-id ${SUBNETS[$NAME-a]} \
    --security-group-ids ${SGS[$NAME]} \
    --iam-instance-profile Name=FirstEc2Profile \
    --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=lab-$NAME-test}]" \
    --query 'Instances[0].InstanceId' --output text)
  INSTANCES[$NAME]=$INST
done

sleep 60

# Each instance 의 private IP
for NAME in prod dev shared; do
  IP=$(aws ec2 describe-instances --instance-ids ${INSTANCES[$NAME]} \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
  echo "$NAME: ${INSTANCES[$NAME]} = $IP"
done
```

## 10. Isolation Test

```bash
# Prod 의 EC2 에서 ping
aws ssm send-command --instance-ids ${INSTANCES[prod]} \
  --document-name AWS-RunShellScript \
  --parameters 'commands=[
    "ping -c 3 -W 2 '$(aws ec2 describe-instances --instance-ids ${INSTANCES[shared]} --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)'",
    "ping -c 3 -W 2 '$(aws ec2 describe-instances --instance-ids ${INSTANCES[dev]} --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)'"
  ]'
# Prod → Shared: OK
# Prod → Dev: 차단 (timeout)

# Dev 의 EC2 에서
aws ssm send-command --instance-ids ${INSTANCES[dev]} \
  --document-name AWS-RunShellScript \
  --parameters 'commands=[
    "ping -c 3 -W 2 '$(aws ec2 describe-instances --instance-ids ${INSTANCES[shared]} --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)'",
    "ping -c 3 -W 2 '$(aws ec2 describe-instances --instance-ids ${INSTANCES[prod]} --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)'"
  ]'
# Dev → Shared: OK
# Dev → Prod: 차단
```

## 11. TGW Route Table 확인

```bash
# Prod RT 의 routes
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $TGW_RT_PROD \
  --filters Name=state,Values=active

# Shared RT 의 routes
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $TGW_RT_SHARED \
  --filters Name=state,Values=active
```

→ Shared RT 는 Prod + Dev 의 CIDR 모두 있음. Prod RT 는 Shared 만.

## 12. Reachability Analyzer (Cross-VPC)

```bash
ENI_PROD=$(aws ec2 describe-instances --instance-ids ${INSTANCES[prod]} \
  --query 'Reservations[0].Instances[0].NetworkInterfaces[0].NetworkInterfaceId' --output text)
ENI_DEV=$(aws ec2 describe-instances --instance-ids ${INSTANCES[dev]} \
  --query 'Reservations[0].Instances[0].NetworkInterfaces[0].NetworkInterfaceId' --output text)

PATH_ID=$(aws ec2 create-network-insights-path \
  --source $ENI_PROD --destination $ENI_DEV \
  --protocol icmp \
  --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text)

ANALYSIS_ID=$(aws ec2 start-network-insights-analysis \
  --network-insights-path-id $PATH_ID \
  --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text)

aws ec2 wait network-insights-analysis-status-succeeded --network-insights-analysis-ids $ANALYSIS_ID

aws ec2 describe-network-insights-analyses --network-insights-analysis-ids $ANALYSIS_ID \
  --query 'NetworkInsightsAnalyses[0].{Found:NetworkPathFound,Explanations:Explanations}'
# Found = false, Explanation 에 TGW RT 의 isolation 표시
```

## 13. Clean up

```bash
# Instances
for NAME in prod dev shared; do
  aws ec2 terminate-instances --instance-ids ${INSTANCES[$NAME]}
done

for NAME in prod dev shared; do
  aws ec2 wait instance-terminated --instance-ids ${INSTANCES[$NAME]}
done

# SG
for NAME in prod dev shared; do
  aws ec2 delete-security-group --group-id ${SGS[$NAME]}
done

# TGW VPC Attachments
for NAME in prod dev shared; do
  aws ec2 delete-transit-gateway-vpc-attachment \
    --transit-gateway-attachment-id ${TGW_ATTS[$NAME]}
done

sleep 60

# TGW Route Tables
for RT in $TGW_RT_PROD $TGW_RT_DEV $TGW_RT_SHARED; do
  aws ec2 delete-transit-gateway-route-table --transit-gateway-route-table-id $RT
done

# TGW
aws ec2 delete-transit-gateway --transit-gateway-id $TGW_ID

# VPCs
for NAME in prod dev shared; do
  for KEY in $NAME-a $NAME-b $NAME-c; do
    aws ec2 delete-subnet --subnet-id ${SUBNETS[$KEY]}
  done
  aws ec2 delete-vpc --vpc-id ${VPCS[$NAME]}
done
```

## 14. 학습 포인트

- TGW 의 hub-spoke topology
- Custom Route Table 의 isolation (Prod ↔ Dev 차단)
- Association + Propagation 의 의미
- VPC RT 의 10.0.0.0/8 → TGW route
- Cross-VPC ping test (private IP)
- Reachability Analyzer 의 자동 path 검증

## 15. 비용 인지

```
TGW Attachment: $0.05/h × 3 attachments × 730h = $109/월
+ DT: $0.02/GB
= 작은 실습이지만 production 큰 비용
```

## 16. 다음

- ▶ [[../vpc]]
- ▶ [[../../alb/alb]]

## 17. 참고

- [[../vpc]]
- [[../02-networking/02-peering-and-transit-gateway]]
- [[01-3-tier-vpc]]
