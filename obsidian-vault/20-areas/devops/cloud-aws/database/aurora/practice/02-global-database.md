---
title: "Aurora 실습 2 — Global Database + Cross-region Failover"
kind: practice
status: current
created_at: 2026-05-22T20:42:00+09:00
tags: [aws, aurora, practice, global-database, dr, failover, multi-region]
home_hub: aurora
---

# 실습 2 — Aurora Global Database + Failover

> 목표: Primary (Seoul) + Secondary (Tokyo) Global Database + Managed Switchover + Unplanned Failover. 60분.

## 1. Primary KMS + Cluster (Seoul)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

KEY_APS2=$(aws kms create-key --description "lab aurora aps2" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-aurora-aps2 --target-key-id $KEY_APS2

# Cluster (Seoul)
aws rds create-db-cluster \
  --db-cluster-identifier lab-aurora-aps2 \
  --engine aurora-postgresql --engine-version 16.2 \
  --master-username lab_admin \
  --manage-master-user-password \
  --database-name labdb \
  --serverless-v2-scaling-configuration "MinCapacity=0.5,MaxCapacity=2" \
  --storage-encrypted --kms-key-id alias/lab-aurora-aps2 \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name lab-db-subnet \
  --backup-retention-period 7 \
  --enable-cloudwatch-logs-exports postgresql

# Writer
aws rds create-db-instance \
  --db-instance-identifier lab-aurora-aps2-writer \
  --db-cluster-identifier lab-aurora-aps2 \
  --engine aurora-postgresql \
  --db-instance-class db.serverless

aws rds wait db-instance-available --db-instance-identifier lab-aurora-aps2-writer
```

## 2. Global Cluster 생성

```bash
aws rds create-global-cluster \
  --global-cluster-identifier lab-global \
  --source-db-cluster-identifier arn:aws:rds:ap-northeast-2:$ACCT:cluster:lab-aurora-aps2
```

→ lab-aurora-aps2 가 primary 로 promoted.

## 3. Tokyo VPC + Subnet Group + SG

```bash
VPC_APN1=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --region ap-northeast-1 --query 'Vpcs[0].VpcId' --output text)

SUBNETS_APN1=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_APN1 \
  --region ap-northeast-1 --query 'Subnets[].SubnetId' --output text))

aws rds create-db-subnet-group \
  --db-subnet-group-name lab-db-subnet \
  --db-subnet-group-description "tokyo" \
  --subnet-ids ${SUBNETS_APN1[@]} \
  --region ap-northeast-1

SG_RDS_APN1=$(aws ec2 create-security-group --vpc-id $VPC_APN1 \
  --group-name lab-rds --description "rds tokyo" \
  --region ap-northeast-1 --query GroupId --output text)
```

## 4. Tokyo KMS

```bash
KEY_APN1=$(aws kms create-key --description "lab aurora apn1" \
  --region ap-northeast-1 --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-aurora-apn1 \
  --target-key-id $KEY_APN1 --region ap-northeast-1
```

## 5. Secondary Cluster (Tokyo)

```bash
aws rds create-db-cluster \
  --db-cluster-identifier lab-aurora-apn1 \
  --global-cluster-identifier lab-global \
  --engine aurora-postgresql \
  --engine-version 16.2 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:ap-northeast-1:$ACCT:key/$KEY_APN1 \
  --vpc-security-group-ids $SG_RDS_APN1 \
  --db-subnet-group-name lab-db-subnet \
  --serverless-v2-scaling-configuration "MinCapacity=0.5,MaxCapacity=2" \
  --region ap-northeast-1

# Secondary instance (Reader only)
aws rds create-db-instance \
  --db-instance-identifier lab-aurora-apn1-reader-1 \
  --db-cluster-identifier lab-aurora-apn1 \
  --engine aurora-postgresql \
  --db-instance-class db.serverless \
  --region ap-northeast-1

echo "Waiting (~15분)..."
aws rds wait db-instance-available --db-instance-identifier lab-aurora-apn1-reader-1 \
  --region ap-northeast-1
```

## 6. Test data + replication 검증

```bash
SECRET_APS2=$(aws rds describe-db-clusters --db-cluster-identifier lab-aurora-aps2 \
  --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text)
PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET_APS2 \
  --query 'SecretString' --output text | jq -r .password)

WRITER_APS2=$(aws rds describe-db-instances --db-instance-identifier lab-aurora-aps2-writer \
  --query 'DBInstances[0].Endpoint.Address' --output text)

READER_APN1=$(aws rds describe-db-instances --db-instance-identifier lab-aurora-apn1-reader-1 \
  --region ap-northeast-1 \
  --query 'DBInstances[0].Endpoint.Address' --output text)

# Write to Seoul
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $WRITER_APS2 -U lab_admin -d labdb -c 'CREATE TABLE events (id serial primary key, msg text, ts timestamptz default now())'\",
    \"for i in {1..100}; do PGPASSWORD=$PASSWORD psql -h $WRITER_APS2 -U lab_admin -d labdb -c \\\"INSERT INTO events (msg) VALUES ('event-\$i')\\\"; done\"
  ]"

# Wait <1초 replication
sleep 5

# Read from Tokyo
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $READER_APN1 -U lab_admin -d labdb -c 'SELECT count(*) FROM events'\"
  ]"
# count = 100 (replicated)

# Write 시도 to Tokyo (실패 — read-only)
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $READER_APN1 -U lab_admin -d labdb -c \\\"INSERT INTO events (msg) VALUES ('from tokyo')\\\"\"
  ]"
# ERROR: cannot execute INSERT in a read-only transaction
```

## 7. Replication Lag metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=lab-aurora-apn1 \
  --region ap-northeast-1 \
  --period 60 --statistics Average,Maximum \
  --start-time $(date -u -v -10M +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time   $(date -u                 +"%Y-%m-%dT%H:%M:%SZ")
# Average < 1000ms (1초)
```

## 8. Managed Planned Switchover (Tokyo → Primary)

```bash
# Source 의 write 잠시 중단 → replication sync → switch
aws rds switchover-global-cluster \
  --global-cluster-identifier lab-global \
  --target-db-cluster-arn arn:aws:rds:ap-northeast-1:$ACCT:cluster:lab-aurora-apn1

# 진행 (~수분)
sleep 180
aws rds describe-global-clusters --global-cluster-identifier lab-global \
  --query 'GlobalClusters[0].GlobalClusterMembers[].{Cluster:DBClusterArn,Writer:IsWriter}'
```

→ Tokyo 가 new primary, Seoul 이 new secondary.

## 9. 새 Primary 에서 write

```bash
WRITER_APN1=$(aws rds describe-db-instances --db-instance-identifier lab-aurora-apn1-reader-1 \
  --region ap-northeast-1 \
  --query 'DBInstances[0].Endpoint.Address' --output text)

# 이제 read-only 가 아님 (primary 됨)
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $WRITER_APN1 -U lab_admin -d labdb -c \\\"INSERT INTO events (msg) VALUES ('from tokyo as primary')\\\"\"
  ]"
```

## 10. Unplanned Failover (재난 시뮬)

```bash
# Switchover back to Seoul (planned)
aws rds switchover-global-cluster \
  --global-cluster-identifier lab-global \
  --target-db-cluster-arn arn:aws:rds:ap-northeast-2:$ACCT:cluster:lab-aurora-aps2

sleep 180

# Unplanned: primary 의 응답 없음 가정
aws rds failover-global-cluster \
  --global-cluster-identifier lab-global \
  --target-db-cluster-arn arn:aws:rds:ap-northeast-1:$ACCT:cluster:lab-aurora-apn1 \
  --allow-data-loss

# RPO < 5초 — 일부 데이터 loss 가능
```

## 11. Route 53 Latency-based Routing

```bash
# Hosted Zone (옵션)
aws route53 create-health-check ...   # primary endpoint
aws route53 create-health-check ...   # secondary endpoint

aws route53 change-resource-record-sets --hosted-zone-id $HZ \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "db.bkurs.com",
          "Type": "CNAME",
          "Region": "ap-northeast-2",
          "SetIdentifier": "seoul",
          "TTL": 30,
          "ResourceRecords": [{ "Value": "lab-aurora-aps2.cluster-xxx.ap-northeast-2.rds.amazonaws.com" }]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "db.bkurs.com",
          "Type": "CNAME",
          "Region": "ap-northeast-1",
          "SetIdentifier": "tokyo",
          "TTL": 30,
          "ResourceRecords": [{ "Value": "lab-aurora-apn1.cluster-xxx.ap-northeast-1.rds.amazonaws.com" }]
        }
      }
    ]
  }'
```

→ Client 의 latency 최소 region 자동 선택.

## 12. Clean up

```bash
# Failover back to Seoul (정리 위해)
aws rds switchover-global-cluster \
  --global-cluster-identifier lab-global \
  --target-db-cluster-arn arn:aws:rds:ap-northeast-2:$ACCT:cluster:lab-aurora-aps2 2>/dev/null
sleep 180

# Remove secondary
aws rds remove-from-global-cluster \
  --global-cluster-identifier lab-global \
  --db-cluster-identifier arn:aws:rds:ap-northeast-1:$ACCT:cluster:lab-aurora-apn1

# Delete Tokyo
aws rds delete-db-instance --db-instance-identifier lab-aurora-apn1-reader-1 \
  --region ap-northeast-1 --skip-final-snapshot
aws rds delete-db-cluster --db-cluster-identifier lab-aurora-apn1 \
  --region ap-northeast-1 --skip-final-snapshot

# Delete Seoul
aws rds delete-db-instance --db-instance-identifier lab-aurora-aps2-writer \
  --skip-final-snapshot
aws rds delete-db-cluster --db-cluster-identifier lab-aurora-aps2 \
  --skip-final-snapshot

# Global cluster
aws rds delete-global-cluster --global-cluster-identifier lab-global

# KMS
for K in $KEY_APS2; do
  aws kms schedule-key-deletion --key-id $K --pending-window-in-days 7
done
aws kms schedule-key-deletion --key-id $KEY_APN1 --pending-window-in-days 7 --region ap-northeast-1
aws kms delete-alias --alias-name alias/lab-aurora-aps2
aws kms delete-alias --alias-name alias/lab-aurora-apn1 --region ap-northeast-1

# Tokyo SG + Subnet Group
aws ec2 delete-security-group --group-id $SG_RDS_APN1 --region ap-northeast-1
aws rds delete-db-subnet-group --db-subnet-group-name lab-db-subnet --region ap-northeast-1
```

## 13. 학습 포인트

- Global Database 의 storage-level replication < 1초
- Managed Switchover (planned, no data loss)
- Unplanned Failover (RPO < 5초, allow-data-loss)
- Secondary 의 read-only enforcement
- Cross-region KMS key 필요 (each region 별)
- Route 53 latency-based routing 의 client routing

## 14. 다음

- ▶ [[../aurora]]
- ▶ [[../../dynamodb/dynamodb]]
- ▶ [[../02-operations/02-global-database-and-dr]]

## 15. 참고

- [[../aurora]]
- [[01-serverless-v2-cluster]]
- [[../02-operations/02-global-database-and-dr]]
