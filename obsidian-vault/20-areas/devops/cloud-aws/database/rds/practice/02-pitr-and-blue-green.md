---
title: "RDS 실습 2 — PITR + Blue-Green Major Version Upgrade"
kind: practice
status: current
created_at: 2026-05-22T20:16:00+09:00
tags: [aws, rds, practice, pitr, blue-green, upgrade]
home_hub: rds
---

# 실습 2 — PITR + Blue-Green Major Upgrade

> 목표: PITR 으로 데이터 복원 + Blue-Green 으로 Postgres 14 → 16 zero-downtime upgrade. 45분.

## 1. Old version DB (Postgres 14)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
# 실습 1 의 SG / Subnet Group / Monitoring Role 재사용

aws rds create-db-instance \
  --db-instance-identifier lab-pg14 \
  --engine postgres --engine-version 14.10 \
  --db-instance-class db.t4g.medium \
  --allocated-storage 50 --storage-type gp3 \
  --storage-encrypted \
  --master-username lab_admin \
  --manage-master-user-password \
  --db-name labdb \
  --multi-az \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name lab-db-subnet \
  --backup-retention-period 7 \
  --enable-cloudwatch-logs-exports postgresql upgrade \
  --deletion-protection

aws rds wait db-instance-available --db-instance-identifier lab-pg14
```

## 2. Test data

```bash
SECRET=$(aws rds describe-db-instances --db-instance-identifier lab-pg14 \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)
PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET \
  --query 'SecretString' --output text | jq -r .password)
ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier lab-pg14 \
  --query 'DBInstances[0].Endpoint.Address' --output text)

aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $ENDPOINT -U lab_admin -d labdb -c 'CREATE TABLE events (id serial primary key, ts timestamptz default now(), name text)'\",
    \"for i in {1..1000}; do PGPASSWORD=$PASSWORD psql -h $ENDPOINT -U lab_admin -d labdb -c \\\"INSERT INTO events (name) VALUES ('event-\$i')\\\"; done\",
    \"PGPASSWORD=$PASSWORD psql -h $ENDPOINT -U lab_admin -d labdb -c 'SELECT count(*) FROM events'\"
  ]"

# 현재 시각 기록 (PITR 용)
TIMESTAMP_BEFORE_DELETE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
echo "PITR target: $TIMESTAMP_BEFORE_DELETE"
sleep 60
```

## 3. "실수" 데이터 삭제

```bash
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $ENDPOINT -U lab_admin -d labdb -c 'DELETE FROM events WHERE id > 500'\",
    \"PGPASSWORD=$PASSWORD psql -h $ENDPOINT -U lab_admin -d labdb -c 'SELECT count(*) FROM events'\"
  ]"
# count = 500 (실수 삭제)
```

## 4. PITR 복원

```bash
# Restorable time range 확인
aws rds describe-db-instances --db-instance-identifier lab-pg14 \
  --query 'DBInstances[0].LatestRestorableTime'

# Restore to specific time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier lab-pg14 \
  --target-db-instance-identifier lab-pg14-restored \
  --restore-time "$TIMESTAMP_BEFORE_DELETE" \
  --db-instance-class db.t4g.medium \
  --no-publicly-accessible \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name lab-db-subnet

aws rds wait db-instance-available --db-instance-identifier lab-pg14-restored
```

## 5. Restored 데이터 확인

```bash
RESTORED_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier lab-pg14-restored \
  --query 'DBInstances[0].Endpoint.Address' --output text)
RESTORED_SECRET=$(aws rds describe-db-instances --db-instance-identifier lab-pg14-restored \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)
RESTORED_PASSWORD=$(aws secretsmanager get-secret-value --secret-id $RESTORED_SECRET \
  --query 'SecretString' --output text | jq -r .password)

aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$RESTORED_PASSWORD psql -h $RESTORED_ENDPOINT -U lab_admin -d labdb -c 'SELECT count(*) FROM events'\"
  ]"
# count = 1000 (복원됨)
```

## 6. Blue-Green Deployment 시작 (PG 14 → 16)

```bash
# 새 Parameter Group (PG 16)
aws rds create-db-parameter-group \
  --db-parameter-group-name lab-pg16-target \
  --db-parameter-group-family postgres16 \
  --description "lab BG target"

# Blue-Green Deployment
aws rds create-blue-green-deployment \
  --blue-green-deployment-name lab-pg-upgrade \
  --source arn:aws:rds:ap-northeast-2:$ACCT:db:lab-pg14 \
  --target-engine-version 16.2 \
  --target-db-parameter-group-name lab-pg16-target \
  --tags Key=Environment,Value=lab Key=Reason,Value=upgrade

BGD_ID=$(aws rds describe-blue-green-deployments \
  --query 'BlueGreenDeployments[?BlueGreenDeploymentName==`lab-pg-upgrade`].BlueGreenDeploymentIdentifier' --output text)
echo "BGD_ID=$BGD_ID"

# 생성 + replication 동기화 (~10-20분)
aws rds wait blue-green-deployment-available --blue-green-deployment-identifier $BGD_ID
```

## 7. Green 검증

```bash
# Green endpoint (별도)
GREEN_ENDPOINT=$(aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier $BGD_ID \
  --query 'BlueGreenDeployments[0].Target.[Endpoint]' --output text)

aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $GREEN_ENDPOINT -U lab_admin -d labdb -c 'SHOW server_version'\",
    \"PGPASSWORD=$PASSWORD psql -h $GREEN_ENDPOINT -U lab_admin -d labdb -c 'SELECT count(*) FROM events'\",
    \"PGPASSWORD=$PASSWORD psql -h $GREEN_ENDPOINT -U lab_admin -d labdb -c 'SELECT MAX(id) FROM events'\"
  ]"
# server_version = 16.2 (upgraded)
# count = 1000 (replicated)
```

## 8. App 호환성 test (선택)

> production 에서는 staging environment 의 app 을 Green endpoint 로 redirect 후 smoke test.

## 9. Switch over

```bash
# Replication catch-up 확인 + switch
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier $BGD_ID \
  --switchover-timeout 300

# Switch 완료 대기 (~수초)
aws rds wait blue-green-deployment-switchover-completed \
  --blue-green-deployment-identifier $BGD_ID

# 이제 lab-pg14 = Postgres 16 (endpoint 같음)
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $ENDPOINT -U lab_admin -d labdb -c 'SHOW server_version'\"
  ]"
# 16.2
```

## 10. 검증

```bash
# Endpoint 같지만 engine 만 16
aws rds describe-db-instances --db-instance-identifier lab-pg14 \
  --query 'DBInstances[0].{Engine:Engine,Version:EngineVersion,Endpoint:Endpoint.Address}'

# Old Blue instance (이름 변경됨)
aws rds describe-db-instances \
  --query 'DBInstances[?DBInstanceIdentifier!=`lab-pg14`&&starts_with(DBInstanceIdentifier,`lab-pg14-`)].[DBInstanceIdentifier,Engine,EngineVersion]'
```

## 11. Rollback 시뮬 (5분 안)

```bash
# Switch 후 5분 안이면 새 Blue-Green 으로 rollback 가능 (Old Blue 가 read-only standby 로 유지)
# 5분 후 자동 stop

aws rds describe-db-instances \
  --query 'DBInstances[?starts_with(DBInstanceIdentifier,`lab-pg14`)]'
```

## 12. Blue-Green 정리

```bash
# Switch 완료 후 BGD 자체 삭제 (Old Blue 도)
aws rds delete-blue-green-deployment --blue-green-deployment-identifier $BGD_ID \
  --delete-target true   # Old Blue 도 같이 삭제
```

## 13. Final Clean up

```bash
aws rds modify-db-instance --db-instance-identifier lab-pg14 \
  --no-deletion-protection --apply-immediately
aws rds delete-db-instance --db-instance-identifier lab-pg14 \
  --skip-final-snapshot --delete-automated-backups

aws rds delete-db-instance --db-instance-identifier lab-pg14-restored \
  --skip-final-snapshot --delete-automated-backups 2>/dev/null

aws rds delete-db-parameter-group --db-parameter-group-name lab-pg16-target

# Secret cleanup
for S in $SECRET $RESTORED_SECRET; do
  aws secretsmanager delete-secret --secret-id $S --force-delete-without-recovery 2>/dev/null
done
```

## 14. 학습 포인트

- PITR 의 1분 granularity (실수 삭제 복구)
- LatestRestorableTime 의 의미
- Blue-Green Deployment (2022+) — zero-downtime major upgrade
- Logical replication (Postgres) / binlog (MySQL)
- Switch over ~수초 (DNS + endpoint swap)
- Old Blue 5분 grace period (rollback)
- delete-target 으로 Old Blue 정리

## 15. 다음

- ▶ [[../rds]]
- ▶ [[../../aurora/aurora]]
- ▶ [[../02-operations/03-migration-and-blue-green]]

## 16. 참고

- [[../rds]]
- [[01-rds-multi-az-postgres]]
- [[../02-operations/01-backup-and-pitr]]
- [[../02-operations/03-migration-and-blue-green]]
