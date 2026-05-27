---
title: "RDS 실습 1 — Multi-AZ Postgres + Read Replica + IAM Auth + Secrets Rotation"
kind: practice
status: current
created_at: 2026-05-22T20:14:00+09:00
tags: [aws, rds, practice, postgres, multi-az, read-replica, iam-auth, secrets]
home_hub: rds
---

# 실습 1 — Multi-AZ Postgres + Read Replica + IAM Auth

> 목표: Multi-AZ Postgres 16 + Read Replica + IAM Database Auth + Secrets Manager rotation + Performance Insights + pgaudit. 60분.

## 1. VPC + Subnet Group

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=($(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[].SubnetId' --output text))

# DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name lab-db-subnet \
  --db-subnet-group-description "lab" \
  --subnet-ids ${SUBNETS[@]}
```

## 2. KMS Key

```bash
KEY_ID=$(aws kms create-key --description "lab rds" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-rds --target-key-id $KEY_ID

# Secrets Manager key 별도
KEY_SECRET=$(aws kms create-key --description "lab secrets" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-secrets --target-key-id $KEY_SECRET
```

## 3. Security Group

```bash
SG_APP=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-app \
  --description "app" --query GroupId --output text)

SG_RDS=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-rds \
  --description "rds" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_RDS \
  --protocol tcp --port 5432 --source-group $SG_APP
```

## 4. Parameter Group (Custom)

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name lab-pg16 \
  --db-parameter-group-family postgres16 \
  --description "lab custom"

aws rds modify-db-parameter-group --db-parameter-group-name lab-pg16 \
  --parameters \
    "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=log_statement,ParameterValue=mod,ApplyMethod=immediate" \
    "ParameterName=shared_preload_libraries,ParameterValue=pgaudit,ApplyMethod=pending-reboot" \
    "ParameterName=pgaudit.log,ParameterValue=write,ddl,role,ApplyMethod=immediate" \
    "ParameterName=work_mem,ParameterValue=4096,ApplyMethod=immediate"
```

## 5. Monitoring Role

```bash
aws iam create-role --role-name rds-monitoring-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"monitoring.rds.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true

aws iam attach-role-policy --role-name rds-monitoring-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole \
  2>/dev/null || true
```

## 6. Primary DB (Multi-AZ)

```bash
aws rds create-db-instance \
  --db-instance-identifier lab-postgres \
  --engine postgres --engine-version 16.2 \
  --db-instance-class db.t4g.medium \
  --allocated-storage 100 --storage-type gp3 \
  --storage-encrypted --kms-key-id alias/lab-rds \
  --master-username lab_admin \
  --manage-master-user-password \
  --master-user-secret-kms-key-id alias/lab-secrets \
  --db-name labdb \
  --multi-az \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name lab-db-subnet \
  --db-parameter-group-name lab-pg16 \
  --backup-retention-period 7 \
  --preferred-backup-window "18:00-19:00" \
  --preferred-maintenance-window "Sun:19:00-Sun:20:00" \
  --enable-iam-database-authentication \
  --enable-performance-insights --performance-insights-retention-period 7 \
  --monitoring-interval 60 --monitoring-role-arn arn:aws:iam::$ACCT:role/rds-monitoring-role \
  --enable-cloudwatch-logs-exports postgresql upgrade \
  --deletion-protection \
  --copy-tags-to-snapshot \
  --tags Key=Environment,Value=lab Key=Service,Value=bkurs

echo "Waiting (~15분)..."
aws rds wait db-instance-available --db-instance-identifier lab-postgres
```

## 7. Read Replica (Cross-AZ)

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier lab-postgres-replica \
  --source-db-instance-identifier lab-postgres \
  --db-instance-class db.t4g.medium \
  --enable-performance-insights \
  --monitoring-interval 60 --monitoring-role-arn arn:aws:iam::$ACCT:role/rds-monitoring-role \
  --tags Key=Environment,Value=lab Key=Role,Value=read-replica

aws rds wait db-instance-available --db-instance-identifier lab-postgres-replica
```

## 8. Reboot for pgaudit (shared_preload_libraries)

```bash
aws rds reboot-db-instance --db-instance-identifier lab-postgres --force-failover
sleep 90
aws rds wait db-instance-available --db-instance-identifier lab-postgres
```

## 9. pgaudit + Schema 생성

```bash
# Master password 조회
SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier lab-postgres \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)
PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET_ARN \
  --query 'SecretString' --output text | jq -r .password)

PRIMARY_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier lab-postgres \
  --query 'DBInstances[0].Endpoint.Address' --output text)

# SSM tunnel 또는 bastion 으로 접근 (실습)
# 또는 같은 SG_APP 에 bastion EC2 띄움

aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"dnf install -y postgresql16\",
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 'CREATE EXTENSION pgaudit'\",
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 'CREATE EXTENSION IF NOT EXISTS pgcrypto'\",
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 'CREATE TABLE users (id serial primary key, email text unique, created_at timestamptz default now())'\",
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c \\\"INSERT INTO users (email) VALUES ('alice@bkurs.com'), ('bob@bkurs.com')\\\"\"
  ]"
```

## 10. IAM Database Auth — user 생성

```bash
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 'CREATE USER iam_user'\",
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 'GRANT rds_iam TO iam_user'\",
    \"PGPASSWORD=$PASSWORD psql -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 'GRANT SELECT ON ALL TABLES IN SCHEMA public TO iam_user'\"
  ]"
```

### IAM Policy (caller role)

```bash
DB_RESOURCE_ID=$(aws rds describe-db-instances --db-instance-identifier lab-postgres \
  --query 'DBInstances[0].DbiResourceId' --output text)

cat > iam-db.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:ap-northeast-2:$ACCT:dbuser:$DB_RESOURCE_ID/iam_user"
    }
  ]
}
EOF

aws iam put-role-policy --role-name $BASTION_ROLE \
  --policy-name rds-iam-connect --policy-document file://iam-db.json
```

### IAM Auth 사용

```bash
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"wget -q https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem -O /tmp/global-bundle.pem\",
    \"TOKEN=\\$(aws rds generate-db-auth-token --hostname $PRIMARY_ENDPOINT --port 5432 --region ap-northeast-2 --username iam_user)\",
    \"PGPASSWORD=\\\$TOKEN psql -h $PRIMARY_ENDPOINT -U iam_user -d labdb -c 'SELECT * FROM users LIMIT 5' --set=sslmode=verify-full --set=sslrootcert=/tmp/global-bundle.pem\"
  ]"
```

## 11. Read Replica 의 read

```bash
REPLICA_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier lab-postgres-replica \
  --query 'DBInstances[0].Endpoint.Address' --output text)

aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $REPLICA_ENDPOINT -U lab_admin -d labdb -c 'SELECT pg_is_in_recovery()'\",
    \"PGPASSWORD=$PASSWORD psql -h $REPLICA_ENDPOINT -U lab_admin -d labdb -c 'SELECT * FROM users LIMIT 5'\",
    \"PGPASSWORD=$PASSWORD psql -h $REPLICA_ENDPOINT -U lab_admin -d labdb -c 'INSERT INTO users (email) VALUES (TEST)'\"
  ]"
# 마지막은 ERROR — read-only
```

## 12. Performance Insights 부하 시뮬

```bash
# pgbench
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD pgbench -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -i -s 10\",
    \"PGPASSWORD=$PASSWORD pgbench -h $PRIMARY_ENDPOINT -U lab_admin -d labdb -c 10 -j 4 -T 120 -r\"
  ]"
```

→ Performance Insights Console 에서 Wait events / Top SQL 확인.

## 13. Secrets Manager Rotation 검증

```bash
# Manual rotation trigger
aws secretsmanager rotate-secret --secret-id $SECRET_ARN

# 새 password
sleep 10
NEW_PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET_ARN \
  --query 'SecretString' --output text | jq -r .password)

[ "$PASSWORD" != "$NEW_PASSWORD" ] && echo "Rotated OK"
```

## 14. CloudWatch alarm

```bash
aws cloudwatch put-metric-alarm --alarm-name lab-rds-cpu \
  --metric-name CPUUtilization --namespace AWS/RDS \
  --statistic Average --period 300 --evaluation-periods 3 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=lab-postgres

aws cloudwatch put-metric-alarm --alarm-name lab-rds-replica-lag \
  --metric-name ReplicaLag --namespace AWS/RDS \
  --statistic Maximum --period 60 --evaluation-periods 3 \
  --threshold 30 --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=lab-postgres-replica
```

## 15. Failover 시뮬

```bash
# Manual failover (Multi-AZ Instance)
aws rds reboot-db-instance --db-instance-identifier lab-postgres --force-failover

# 대기 (~1-2분)
aws rds wait db-instance-available --db-instance-identifier lab-postgres

# Endpoint 같지만 standby 가 primary 가 됨 (DNS 자동 변경)
```

## 16. Clean up

```bash
# Deletion Protection 해제
aws rds modify-db-instance --db-instance-identifier lab-postgres \
  --no-deletion-protection --apply-immediately

# Replica 먼저
aws rds delete-db-instance --db-instance-identifier lab-postgres-replica \
  --skip-final-snapshot --delete-automated-backups

# Primary
aws rds delete-db-instance --db-instance-identifier lab-postgres \
  --skip-final-snapshot --delete-automated-backups

aws rds wait db-instance-deleted --db-instance-identifier lab-postgres

# Subnet Group / Parameter Group
aws rds delete-db-subnet-group --db-subnet-group-name lab-db-subnet
aws rds delete-db-parameter-group --db-parameter-group-name lab-pg16

# Secrets
aws secretsmanager delete-secret --secret-id $SECRET_ARN --force-delete-without-recovery

# SG
aws ec2 delete-security-group --group-id $SG_RDS
aws ec2 delete-security-group --group-id $SG_APP

# KMS keys
for K in $KEY_ID $KEY_SECRET; do
  aws kms schedule-key-deletion --key-id $K --pending-window-in-days 7
done
aws kms delete-alias --alias-name alias/lab-rds
aws kms delete-alias --alias-name alias/lab-secrets

# Monitoring Role (재사용 시 keep)
```

## 17. 학습 포인트

- Multi-AZ Instance (synchronous standby)
- Read Replica (async)
- Parameter Group + pgaudit (shared_preload_libraries → reboot 필요)
- IAM Database Authentication (15분 token)
- Secrets Manager managed master password + auto rotation
- Performance Insights wait events
- Enhanced Monitoring 60s
- pgbench 부하 + Performance analysis

## 18. 다음

- ▶ [[02-pitr-and-blue-green]]
- ▶ [[../03-security/02-encryption-and-secrets]]
- ▶ [[../03-security/03-audit-and-compliance]]

## 19. 참고

- [[../rds]]
- [[../01-fundamentals/03-multi-az-and-read-replica]]
- [[../03-security/01-network-and-iam]]
