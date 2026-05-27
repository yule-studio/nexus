---
title: "NLB 실습 1 — TCP Load Balance + Postgres + EIP"
kind: practice
status: current
created_at: 2026-05-23T10:42:00+09:00
tags: [aws, nlb, practice, tcp, postgres, eip, static-ip]
how_hub: nlb
---

# 실습 1 — NLB TCP + Postgres

> 목표: NLB TCP listener + 3 EC2 (Postgres) + EIP static IP + Cross-zone + Source IP 보존. 60분.

## 1. VPC + SG

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[?MapPublicIpOnLaunch==`true`]|[0:3].SubnetId' --output text))

# DB SG
SG_DB=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-db \
  --description "DB" --query GroupId --output text)
# NLB 의 IP 보존이면 client IP 가 진짜 client → 0.0.0.0/0 (실습)
aws ec2 authorize-security-group-ingress --group-id $SG_DB \
  --protocol tcp --port 5432 --cidr 0.0.0.0/0
```

## 2. EIP 3 개 (per AZ)

```bash
EIP_A=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
EIP_B=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
EIP_C=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)

aws ec2 describe-addresses --allocation-ids $EIP_A $EIP_B $EIP_C \
  --query 'Addresses[].{Alloc:AllocationId,IP:PublicIp}'
```

## 3. NLB 생성

```bash
NLB_ARN=$(aws elbv2 create-load-balancer --name lab-postgres-nlb \
  --type network --scheme internet-facing \
  --subnet-mappings \
    "SubnetId=${SUBNETS[0]},AllocationId=$EIP_A" \
    "SubnetId=${SUBNETS[1]},AllocationId=$EIP_B" \
    "SubnetId=${SUBNETS[2]},AllocationId=$EIP_C" \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

NLB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $NLB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

# Cross-zone enable
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true

echo "NLB DNS: $NLB_DNS"
echo "NLB EIPs: $(aws ec2 describe-addresses --allocation-ids $EIP_A $EIP_B $EIP_C --query 'Addresses[].PublicIp' --output text)"
```

## 4. Target Group (TCP / IP target / preserve_client_ip)

```bash
TG_ARN=$(aws elbv2 create-target-group --name lab-postgres-tg \
  --protocol TCP --port 5432 \
  --vpc-id $VPC --target-type ip \
  --health-check-protocol TCP --health-check-port 5432 \
  --healthy-threshold-count 3 --unhealthy-threshold-count 3 \
  --health-check-interval-seconds 30 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Source IP 보존
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes \
    Key=preserve_client_ip.enabled,Value=true \
    Key=proxy_protocol_v2.enabled,Value=false \
    Key=deregistration_delay.timeout_seconds,Value=30
```

## 5. 3 Postgres EC2 (per AZ)

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y postgresql16-server postgresql16
postgresql-setup --initdb

# Listen all + md5 auth
sed -i "s/^#listen_addresses.*/listen_addresses = '*'/" /var/lib/pgsql/data/postgresql.conf
echo "host all all 0.0.0.0/0 md5" >> /var/lib/pgsql/data/pg_hba.conf

systemctl enable --now postgresql

# Create user / DB
sudo -u postgres psql <<SQL
CREATE USER lab_user WITH PASSWORD 'lab_password';
CREATE DATABASE lab_db OWNER lab_user;
CREATE TABLE IF NOT EXISTS visits (ts timestamptz default now(), source_ip text, hostname text);
SQL
EOF

declare -A INSTANCES

for i in 0 1 2; do
  AZ_LETTER=$(echo "a b c" | cut -d' ' -f$((i+1)))
  INST=$(aws ec2 run-instances --image-id $AMI --instance-type t4g.small \
    --subnet-id ${SUBNETS[$i]} --security-group-ids $SG_DB \
    --iam-instance-profile Name=FirstEc2Profile \
    --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
    --user-data file://userdata.sh \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=lab-pg-$AZ_LETTER}]" \
    --query 'Instances[0].InstanceId' --output text)
  INSTANCES[$AZ_LETTER]=$INST
done

sleep 180  # Postgres setup

# Register IP targets
for AZ_LETTER in a b c; do
  IP=$(aws ec2 describe-instances --instance-ids ${INSTANCES[$AZ_LETTER]} \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
  aws elbv2 register-targets --target-group-arn $TG_ARN \
    --targets Id=$IP,Port=5432
done

# 의 Listener
aws elbv2 create-listener --load-balancer-arn $NLB_ARN \
  --protocol TCP --port 5432 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

sleep 60
```

## 6. 검증

```bash
# Healthy targets
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].{IP:Target.Id,State:TargetHealth.State}'
# 3 healthy

# Postgres connection via NLB DNS
PGPASSWORD=lab_password psql -h $NLB_DNS -U lab_user -d lab_db -c "SELECT version()"

# Static IP 활용 (EIP 직접)
EIP_A_IP=$(aws ec2 describe-addresses --allocation-ids $EIP_A --query 'Addresses[0].PublicIp' --output text)
PGPASSWORD=lab_password psql -h $EIP_A_IP -U lab_user -d lab_db -c "SELECT version()"
```

## 7. Load Balance 검증

```bash
# 30 conn → 3 instance 로 분산
for i in $(seq 1 30); do
  PGPASSWORD=lab_password psql -h $NLB_DNS -U lab_user -d lab_db -c "
    INSERT INTO visits (source_ip, hostname) VALUES (
      (SELECT split_part(client_addr::text, '/', 1) FROM pg_stat_activity WHERE pid=pg_backend_pid()),
      inet_server_addr()::text
    )
  " &
done
wait

# 분포 확인
for AZ_LETTER in a b c; do
  IP=$(aws ec2 describe-instances --instance-ids ${INSTANCES[$AZ_LETTER]} \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
  COUNT=$(PGPASSWORD=lab_password psql -h $IP -U lab_user -d lab_db -tA -c "SELECT count(*) FROM visits WHERE hostname LIKE '$IP%'")
  echo "AZ-$AZ_LETTER ($IP): $COUNT visits"
done
```

→ Cross-zone ON 이면 균등 (~10 each). OFF 면 client connection 의 AZ 분포 따라.

## 8. Source IP 보존 검증

```bash
# Local IP 확인
MY_IP=$(curl -s https://checkip.amazonaws.com)

# DB 안에서 client_addr 확인
PGPASSWORD=lab_password psql -h $NLB_DNS -U lab_user -d lab_db -c "
  SELECT client_addr, client_port FROM pg_stat_activity WHERE usename='lab_user' AND state='active'
"
# client_addr = $MY_IP (보존됨, NLB IP 아님)
```

## 9. AZ Failover 시뮬

```bash
# AZ-a 의 EC2 stop
aws ec2 stop-instances --instance-ids ${INSTANCES[a]}

sleep 60

# Health 확인
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].{IP:Target.Id,State:TargetHealth.State}'
# AZ-a unhealthy, AZ-b/c healthy

# Connection 여전히 가능 (cross-zone ON)
PGPASSWORD=lab_password psql -h $NLB_DNS -U lab_user -d lab_db -c "SELECT inet_server_addr()"
# AZ-b 또는 AZ-c 의 IP
```

## 10. Cross-Zone OFF 비교

```bash
# Cross-zone OFF
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=false

# AZ-a 의 EC2 down 인 채로 — EIP_A 로 connection 시도
PGPASSWORD=lab_password psql -h $EIP_A_IP -U lab_user -d lab_db -c "SELECT 1"
# Connection refused (AZ-a 의 target 모두 unhealthy)

# 다른 EIP 는 OK
PGPASSWORD=lab_password psql -h $(aws ec2 describe-addresses --allocation-ids $EIP_B --query 'Addresses[0].PublicIp' --output text) \
  -U lab_user -d lab_db -c "SELECT 1"
```

→ Cross-zone OFF = AZ-specific. Cross-zone ON = 자동 failover.

```bash
# 복원
aws ec2 start-instances --instance-ids ${INSTANCES[a]}
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

## 11. CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/NetworkELB \
  --metric-name ActiveFlowCount \
  --dimensions Name=LoadBalancer,Value=$(echo $NLB_ARN | awk -F'/' '{print "net/"$3"/"$4}') \
  --period 60 --statistics Average,Maximum \
  --start-time $(date -u -v -10M +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")

aws cloudwatch get-metric-statistics --namespace AWS/NetworkELB \
  --metric-name NewFlowCount \
  --dimensions Name=LoadBalancer,Value=$(echo $NLB_ARN | awk -F'/' '{print "net/"$3"/"$4}') \
  --period 60 --statistics Sum
```

## 12. Clean up

```bash
# Listener
for L in $(aws elbv2 describe-listeners --load-balancer-arn $NLB_ARN --query 'Listeners[].ListenerArn' --output text); do
  aws elbv2 delete-listener --listener-arn $L
done

# NLB
aws elbv2 delete-load-balancer --load-balancer-arn $NLB_ARN
sleep 30

# TG
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# EC2
for AZ_LETTER in a b c; do
  aws ec2 terminate-instances --instance-ids ${INSTANCES[$AZ_LETTER]}
done

for AZ_LETTER in a b c; do
  aws ec2 wait instance-terminated --instance-ids ${INSTANCES[$AZ_LETTER]}
done

# EIP
for EIP in $EIP_A $EIP_B $EIP_C; do
  aws ec2 release-address --allocation-id $EIP
done

# SG
aws ec2 delete-security-group --group-id $SG_DB
```

## 13. 학습 포인트

- NLB + EIP per AZ (Static IP)
- TCP listener 의 5432 forward
- IP Target + preserve_client_ip
- Source IP 보존 검증 (pg_stat_activity.client_addr)
- Cross-zone ON vs OFF 의 trade-off
- AZ Failover (cross-zone ON 시 자동)
- Health Check (TCP 5432)

## 14. 다음

- ▶ [[02-nlb-to-alb-chain]]
- ▶ [[../02-features/01-tls-and-mtls]]
- ▶ [[../02-features/02-privatelink-endpoint-service]]

## 15. 참고

- [[../nlb]]
- [[../01-fundamentals/01-listener-and-target-group]]
- [[../01-fundamentals/02-static-ip-and-cross-zone]]
