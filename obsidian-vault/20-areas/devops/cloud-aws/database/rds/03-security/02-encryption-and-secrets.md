---
title: "RDS 03-2 — Encryption (KMS at-rest / TLS in-transit) + Secrets Manager"
kind: knowledge
status: current
created_at: 2026-05-22T20:08:00+09:00
tags: [aws, rds, security, encryption, kms, secrets-manager, rotation]
home_hub: rds
---

# RDS Encryption + Secrets

> At-rest (KMS) + in-transit (TLS) + Secrets Manager 자동 rotation (30d). CMK + Permission Boundary.

## 1. Encryption Layer

```
At Rest (KMS):    storage, snapshot, automated backup, replica
In Transit (TLS): client ↔ DB (port 5432/3306)
Application:     PII column 추가 암호화 (옵션)
```

## 2. At-rest Encryption

```bash
# 생성 시
aws rds create-db-instance \
  --storage-encrypted \
  --kms-key-id alias/bkurs-rds \
  ...
```

> **생성 후 변경 불가** — encryption 활성/비활성 변경 시 snapshot + new instance 필요.

```bash
# Snapshot copy 시 변경
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier bkurs-snap \
  --target-db-snapshot-identifier bkurs-snap-encrypted \
  --kms-key-id alias/bkurs-rds-new
```

## 3. Default Encryption (account-wide)

```bash
# EC2/EBS 와 다르게 RDS 는 default encryption 없음 — instance 마다 명시
# 그러나 AWS Config rule 로 강제 가능
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "rds-storage-encrypted",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "RDS_STORAGE_ENCRYPTED"
  }
}'
```

## 4. KMS Key Policy 권장

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableAdmin",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCT:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowRDSUse",
      "Effect": "Allow",
      "Principal": { "AWS": "*" },
      "Action": [
        "kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey*",
        "kms:ReEncrypt*", "kms:CreateGrant", "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "kms:ViaService": "rds.ap-northeast-2.amazonaws.com" },
        "StringEquals": { "kms:CallerAccount": "ACCT" }
      }
    }
  ]
}
```

## 5. TLS in transit

```bash
# Parameter Group
"ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"

# Postgres client
psql "host=$RDS sslmode=verify-full sslrootcert=global-bundle.pem dbname=bkurs user=admin"

# MySQL client
mysql -h $RDS --ssl-ca=global-bundle.pem --ssl-mode=VERIFY_IDENTITY
```

### CA Certificate 자동 rotation
```bash
# Certificate 만료 알림 — CloudWatch
aws rds describe-pending-maintenance-actions \
  --filters Name=db-instance-id,Values=arn:rds:...

# Auto Apply (또는 maintenance window 활용)
aws rds modify-certificates --certificate-identifier rds-ca-rsa2048-g1
```

## 6. Encrypted Column (application level)

PII column 만 추가 암호화:
```python
# Python
from cryptography.fernet import Fernet
key = boto3_secrets.get("encryption-key")
f = Fernet(key)
encrypted_email = f.encrypt(b"user@example.com")

# DB store as bytea / varbinary
```

또는 KMS Encryption SDK (Envelope Encryption).

## 7. Secrets Manager — Master Password 관리

```bash
# 자동 (modern)
aws rds modify-db-instance --db-instance-identifier bkurs-prod \
  --manage-master-user-password \
  --master-user-secret-kms-key-id alias/bkurs-secrets
```

Secret 의 자동 이름: `rds!cluster-xxx` 또는 `rds!db-xxx`.

```bash
SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier bkurs-prod \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)

# Password 조회
aws secretsmanager get-secret-value --secret-id $SECRET_ARN \
  --query 'SecretString' --output text | jq -r .password
```

## 8. Auto Rotation 30일

자동 활성 (RDS managed secret):
```bash
aws secretsmanager describe-secret --secret-id $SECRET_ARN \
  --query '{Rotation:RotationEnabled,Rule:RotationRules,Last:LastRotatedDate}'
```

→ 매 30일 자동 rotate. application 은 secret manager 에서 fresh password 매번.

## 9. Application 의 password 활용

### Lambda
```python
import boto3, json
sm = boto3.client("secretsmanager")
secret_value = sm.get_secret_value(SecretId="rds!db-xxx")["SecretString"]
secret = json.loads(secret_value)
conn = psycopg2.connect(
    host=secret["host"], port=secret["port"],
    user=secret["username"], password=secret["password"],
    dbname=secret["dbname"]
)
```

### Spring Boot
```yaml
spring:
  cloud:
    aws:
      secretsmanager:
        enabled: true
        prefix: /secret
        defaultContext: rds-bkurs

# application.yml 의 ${rds-bkurs/password} 자동 resolve
```

### Powertools (Python)
```python
from aws_lambda_powertools.utilities.parameters import get_secret

@event_source
def handler(event, context):
    db = get_secret("rds!db-xxx", max_age=300, transform="json")
    conn = psycopg2.connect(**db)
```

## 10. Rotation Lambda (custom)

RDS Postgres / MySQL 자동 rotation Lambda 가 미리 있음 (`SecretsManagerRDSPostgreSQLRotationSingleUser`).

```
1. AWSPENDING — 새 password 생성
2. createSecret
3. setSecret — DB ALTER USER
4. testSecret — 새 password 로 connect
5. finishSecret — AWSCURRENT 로 promote
```

[[../../storage/s3/04-security/01-encryption-sse-kms-c-bucket-keys]] §rotation 참조.

## 11. Cross-account Encrypted Snapshot 공유

```bash
# 1. Snapshot share
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier bkurs-snap \
  --attribute-name restore \
  --values-to-add "OTHER_ACCT"

# 2. KMS grant
aws kms create-grant --key-id alias/bkurs-rds \
  --grantee-principal arn:aws:iam::OTHER_ACCT:role/ImportRole \
  --operations Decrypt CreateGrant DescribeKey

# 3. Other account 의 IAM
{
  "Effect": "Allow",
  "Action": ["kms:Decrypt","kms:CreateGrant"],
  "Resource": "arn:aws:kms:ap-northeast-2:ACCT:key/abc"
}
```

## 12. SSL Mode 비교

| Mode | 의미 |
| --- | --- |
| `disable` | 평문 |
| `allow` | 가능하면 |
| `prefer` | 우선 (default) |
| `require` | 필수 (no verify) |
| `verify-ca` | CA 검증 |
| `verify-full` | CA + hostname ★ |

→ **production = `verify-full`**.

## 13. Secrets Manager 비용

```
$0.40 / secret / 월
+ $0.05 / 10K API call
```

> Auto rotation 무료. 일반 RDS 1 secret = $0.40/월.

## 14. Backup 의 Encryption

```
Automated backup:  source instance 의 KMS key 자동
Manual snapshot:   source 의 key
Cross-region copy: 새 region 의 key 명시 (re-encrypt)
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Encryption 생성 후 변경 시도 | snapshot + restore + new key |
| KMS key 권한 누락 → restore 실패 | kms:Decrypt + grant |
| TLS 강제 안 함 | rds.force_ssl=1 |
| Master password env 평문 | Secrets Manager |
| Rotation 후 application 재시작 안 함 | secret cache TTL |
| Cross-account snapshot 의 KMS grant 누락 | 둘 다 |

## 16. 다음

- ▶ [[03-audit-and-compliance]]
- ▶ [[../practice/01-rds-multi-az-postgres]]
- ▶ [[../../../security/kms/kms]]
- ▶ [[../../../security/secrets-manager/secrets-manager]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [RDS Encryption](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html) |
| [Secrets Manager RDS](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) |

## 18. 참고

- [[../rds]]
- [[01-network-and-iam]]
