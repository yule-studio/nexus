---
title: "Aurora 실습 1 — Serverless v2 Cluster + RDS Proxy + Lambda + Scaling"
kind: practice
status: current
created_at: 2026-05-22T20:40:00+09:00
tags: [aws, aurora, practice, serverless, rds-proxy, lambda, scaling]
home_hub: aurora
---

# 실습 1 — Aurora Serverless v2 + RDS Proxy + Lambda

> 목표: Aurora Serverless v2 + RDS Proxy + Lambda 연결 + 부하 시뮬 → ACU 자동 scale 관찰. 60분.

## 1. KMS + Subnet Group + SG

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

KEY_ID=$(aws kms create-key --description "lab aurora" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-aurora --target-key-id $KEY_ID

# 실습 1 의 SG_APP, SG_RDS, lab-db-subnet 재사용 또는 새로
```

## 2. Aurora Serverless v2 Cluster

```bash
aws rds create-db-cluster \
  --db-cluster-identifier lab-serverless \
  --engine aurora-postgresql \
  --engine-version 16.2 \
  --master-username lab_admin \
  --manage-master-user-password \
  --database-name labdb \
  --serverless-v2-scaling-configuration "MinCapacity=0.5,MaxCapacity=4" \
  --storage-encrypted --kms-key-id alias/lab-aurora \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name lab-db-subnet \
  --backup-retention-period 7 \
  --enable-iam-database-authentication \
  --enable-cloudwatch-logs-exports postgresql \
  --deletion-protection

# Writer (Serverless v2)
aws rds create-db-instance \
  --db-instance-identifier lab-serverless-writer \
  --db-cluster-identifier lab-serverless \
  --engine aurora-postgresql \
  --db-instance-class db.serverless \
  --enable-performance-insights

# Reader (Serverless v2)
aws rds create-db-instance \
  --db-instance-identifier lab-serverless-reader \
  --db-cluster-identifier lab-serverless \
  --engine aurora-postgresql \
  --db-instance-class db.serverless

echo "Waiting (~15분)..."
aws rds wait db-instance-available --db-instance-identifier lab-serverless-writer
aws rds wait db-instance-available --db-instance-identifier lab-serverless-reader
```

## 3. RDS Proxy

```bash
# Proxy IAM Role
aws iam create-role --role-name lab-rds-proxy-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"rds.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name lab-rds-proxy-role --policy-name p --policy-document "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [{
    \"Effect\": \"Allow\",
    \"Action\": [\"secretsmanager:GetSecretValue\",\"secretsmanager:DescribeSecret\"],
    \"Resource\": \"arn:aws:secretsmanager:ap-northeast-2:$ACCT:secret:rds!*\"
  }, {
    \"Effect\": \"Allow\",
    \"Action\": [\"kms:Decrypt\"],
    \"Resource\": \"*\",
    \"Condition\": { \"StringEquals\": { \"kms:ViaService\": \"secretsmanager.ap-northeast-2.amazonaws.com\" } }
  }]
}"

SECRET_ARN=$(aws rds describe-db-clusters --db-cluster-identifier lab-serverless \
  --query 'DBClusters[0].MasterUserSecret.SecretArn' --output text)

aws rds create-db-proxy --db-proxy-name lab-aurora-proxy \
  --engine-family POSTGRESQL \
  --role-arn arn:aws:iam::$ACCT:role/lab-rds-proxy-role \
  --vpc-subnet-ids ${SUBNETS[@]} \
  --vpc-security-group-ids $SG_RDS \
  --auth "Description=lab,AuthScheme=SECRETS,SecretArn=$SECRET_ARN,IAMAuth=DISABLED" \
  --require-tls

# Target
aws rds register-db-proxy-targets --db-proxy-name lab-aurora-proxy \
  --db-cluster-identifiers lab-serverless

aws rds wait db-proxy-target-available --db-proxy-name lab-aurora-proxy
```

## 4. Schema + data

```bash
PASSWORD=$(aws secretsmanager get-secret-value --secret-id $SECRET_ARN \
  --query 'SecretString' --output text | jq -r .password)

PROXY_ENDPOINT=$(aws rds describe-db-proxies --db-proxy-name lab-aurora-proxy \
  --query 'DBProxies[0].Endpoint' --output text)

# Schema 생성 (SSM tunnel 또는 bastion)
aws ssm send-command --instance-ids $BASTION_ID \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"PGPASSWORD=$PASSWORD psql -h $PROXY_ENDPOINT -U lab_admin -d labdb -c 'CREATE TABLE jobs (id serial primary key, created_at timestamptz default now(), payload jsonb)'\",
    \"PGPASSWORD=$PASSWORD psql -h $PROXY_ENDPOINT -U lab_admin -d labdb -c 'CREATE INDEX idx_jobs_created ON jobs(created_at)'\"
  ]"
```

## 5. Lambda (Aurora + RDS Proxy)

```bash
# Lambda IAM Role + VPC permission
aws iam create-role --role-name lab-lambda-aurora-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name lab-lambda-aurora-role --policy-name p --policy-document "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [
    { \"Effect\": \"Allow\", \"Action\": \"logs:*\", \"Resource\": \"*\" },
    { \"Effect\": \"Allow\", \"Action\": [\"ec2:CreateNetworkInterface\",\"ec2:DescribeNetworkInterfaces\",\"ec2:DeleteNetworkInterface\"], \"Resource\": \"*\" },
    { \"Effect\": \"Allow\", \"Action\": [\"secretsmanager:GetSecretValue\"], \"Resource\": \"$SECRET_ARN\" }
  ]
}"
```

```python
# lambda_app.py
import boto3, json, os, psycopg2
from datetime import datetime

sm = boto3.client("secretsmanager")
SECRET = json.loads(sm.get_secret_value(SecretId=os.environ["SECRET_ARN"])["SecretString"])

# Connection per Lambda invocation (RDS Proxy 가 pool)
def get_conn():
    return psycopg2.connect(
        host=os.environ["PROXY_ENDPOINT"],
        port=5432,
        user=SECRET["username"],
        password=SECRET["password"],
        dbname="labdb",
        connect_timeout=5
    )

def lambda_handler(event, context):
    n = event.get("count", 100)
    payload = event.get("payload", {})
    
    with get_conn() as conn:
        with conn.cursor() as cur:
            for i in range(n):
                cur.execute(
                    "INSERT INTO jobs (payload) VALUES (%s)",
                    [json.dumps({**payload, "i": i, "ts": datetime.utcnow().isoformat()})]
                )
        conn.commit()
    
    return {"statusCode": 200, "body": json.dumps({"inserted": n})}
```

```bash
mkdir -p lambda-pkg
pip install --platform manylinux2014_aarch64 --only-binary=:all: \
  --target lambda-pkg psycopg2-binary
cp lambda_app.py lambda-pkg/
cd lambda-pkg && zip -r ../lambda-app.zip . && cd ..

aws lambda create-function --function-name lab-aurora-insert \
  --runtime python3.12 --architectures arm64 \
  --handler lambda_app.lambda_handler \
  --role arn:aws:iam::$ACCT:role/lab-lambda-aurora-role \
  --zip-file fileb://lambda-app.zip \
  --memory-size 512 --timeout 60 \
  --environment "Variables={PROXY_ENDPOINT=$PROXY_ENDPOINT,SECRET_ARN=$SECRET_ARN}" \
  --vpc-config "SubnetIds=$(echo ${SUBNETS[@]} | tr ' ' ','),SecurityGroupIds=$SG_APP" \
  --tracing-config Mode=Active
```

## 6. ACU baseline (idle)

```bash
sleep 60
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name ServerlessDatabaseCapacity \
  --dimensions Name=DBClusterIdentifier,Value=lab-serverless \
  --period 60 --statistics Average \
  --start-time $(date -u -v -10M +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time   $(date -u                  +"%Y-%m-%dT%H:%M:%SZ")
# Average 0.5 (idle min)
```

## 7. 부하 시뮬

```bash
# Concurrent Lambda invoke (100 parallel × 1000 rows = 100K)
for i in $(seq 1 100); do
  aws lambda invoke --function-name lab-aurora-insert \
    --payload "{\"count\":1000,\"payload\":{\"batch\":$i}}" \
    --cli-binary-format raw-in-base64-out \
    --invocation-type Event \
    /tmp/out-$i.json &
done
wait

# Lambda Throttle / ACU scaling 관찰
sleep 60
```

## 8. ACU + Connection 관찰

```bash
# Real-time
watch -n 10 "aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name ServerlessDatabaseCapacity \
  --dimensions Name=DBClusterIdentifier,Value=lab-serverless \
  --period 60 --statistics Maximum \
  --start-time \$(date -u -v -5M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   \$(date -u                  +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Datapoints[].[Timestamp,Maximum]' --output table"

# Connection count
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=lab-serverless-writer \
  --period 60 --statistics Maximum
```

전형적 결과:
```
ACU: 0.5 → 1 → 2 → 4 (max 도달)
Connections: 100 → 200 → 400 (RDS Proxy 가 pool)
```

## 9. RDS Proxy 의 효과

```bash
# Direct vs Proxy connection count
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=ProxyName,Value=lab-aurora-proxy \
  --period 60 --statistics Maximum

aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name ClientConnections \
  --dimensions Name=ProxyName,Value=lab-aurora-proxy \
  --period 60 --statistics Maximum
```

| Metric | 의미 |
| --- | --- |
| `DatabaseConnections` (to Proxy) | Proxy → DB connection (작음) |
| `ClientConnections` (to Proxy) | Lambda → Proxy connection (큼) |

→ Proxy 가 connection pool — DB connection count ↓.

## 10. Aurora Clone — Dev environment

```bash
aws rds restore-db-cluster-to-point-in-time \
  --db-cluster-identifier lab-serverless-clone \
  --source-db-cluster-identifier lab-serverless \
  --restore-type copy-on-write \
  --use-latest-restorable-time \
  --serverless-v2-scaling-configuration "MinCapacity=0.5,MaxCapacity=1" \
  --vpc-security-group-ids $SG_RDS \
  --db-subnet-group-name lab-db-subnet

aws rds create-db-instance \
  --db-instance-identifier lab-clone-writer \
  --db-cluster-identifier lab-serverless-clone \
  --engine aurora-postgresql \
  --db-instance-class db.serverless
```

→ Production data 의 zero-copy clone (~수분, no storage cost for unchanged blocks).

## 11. Clean up

```bash
# Deletion Protection
aws rds modify-db-cluster --db-cluster-identifier lab-serverless \
  --no-deletion-protection --apply-immediately

aws rds modify-db-cluster --db-cluster-identifier lab-serverless-clone \
  --no-deletion-protection --apply-immediately 2>/dev/null

# Lambda
aws lambda delete-function --function-name lab-aurora-insert
aws iam delete-role-policy --role-name lab-lambda-aurora-role --policy-name p
aws iam delete-role --role-name lab-lambda-aurora-role

# Proxy
aws rds delete-db-proxy --db-proxy-name lab-aurora-proxy

# Instances
for I in lab-serverless-writer lab-serverless-reader lab-clone-writer; do
  aws rds delete-db-instance --db-instance-identifier $I \
    --skip-final-snapshot --delete-automated-backups 2>/dev/null
done

aws rds wait db-instance-deleted --db-instance-identifier lab-serverless-writer 2>/dev/null
aws rds wait db-instance-deleted --db-instance-identifier lab-serverless-reader 2>/dev/null

# Clusters
for C in lab-serverless lab-serverless-clone; do
  aws rds delete-db-cluster --db-cluster-identifier $C \
    --skip-final-snapshot --delete-automated-backups 2>/dev/null
done

# IAM / KMS
aws iam delete-role-policy --role-name lab-rds-proxy-role --policy-name p
aws iam delete-role --role-name lab-rds-proxy-role
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/lab-aurora
```

## 12. 학습 포인트

- Aurora Serverless v2 의 0.5 ACU min (no cold start)
- ACU 의 초 단위 scaling
- RDS Proxy 의 connection pooling (Lambda + Aurora 표준)
- `IAMAuth=DISABLED` + Secrets Manager
- Aurora Clone 의 zero-copy (zero downtime dev env)
- ServerlessDatabaseCapacity + ACUUtilization metric
- Multi-Lambda concurrent → connection 폭주 방지

## 13. 다음

- ▶ [[02-global-database]]
- ▶ [[../01-fundamentals/03-serverless-v2]]
- ▶ [[../02-operations/01-cluster-and-endpoint]]

## 14. 참고

- [[../aurora]]
- [[../01-fundamentals/03-serverless-v2]]
- [[../02-operations/03-backup-and-blue-green]]
