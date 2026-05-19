---
title: "AWS Secrets Manager"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:30:00+09:00
tags:
  - aws
  - security
  - secrets
---

# AWS Secrets Manager

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Secrets Manager 개념 + rotation |

**[[security|↑ Security]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**비밀 (DB password / API key / 인증서) 의 매니지드 저장소** + 자동 rotation + IAM access + KMS 암호화.

---

## 2. 왜

- 환경변수 / 코드 / S3 에 password = leak
- Secrets Manager = IAM + KMS + audit + rotation 통합
- RDS / DocumentDB 등은 native rotation 통합

대안:
- **SSM Parameter Store (SecureString)** — 더 싸고 간단, rotation 수동
- **HashiCorp Vault** — 복잡 / multi-cloud / 동적 credentials
- **External Secrets Operator** (K8s) — secret 끌어오기

---

## 3. Secrets Manager vs SSM Parameter Store

| | Secrets Manager | Parameter Store |
| --- | --- | --- |
| 자동 rotation | ✅ | X (수동) |
| 가격 | $0.40 / secret·월 + $0.05/10K | 무료 + advanced $0.05/parameter·월 |
| 통합 | RDS / DocumentDB 자동 | 폭넓음 |
| 버전 관리 | ✅ | ✅ |
| 크기 한계 | 64 KB | 4 KB / 8 KB (advanced) |
| 권장 | 운영 password | config / 일반 |

→ 운영 DB password = Secrets Manager. 일반 config = Parameter Store.

---

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Secret** | 1 개의 비밀 (이름 + value + metadata) |
| **Version** | 같은 secret 의 N 버전 (rotation) |
| **VersionStage** | `AWSCURRENT` / `AWSPENDING` / `AWSPREVIOUS` |
| **Rotation** | Lambda 가 비밀 자동 교체 |
| **Resource Policy** | secret 의 access 정책 |

---

## 5. 설치 / 사용

### 5.1 CLI

```bash
# 생성
aws secretsmanager create-secret \
  --name myapp/db/password \
  --secret-string '{"username":"app","password":"random-strong"}' \
  --kms-key-id alias/myapp

# 조회
aws secretsmanager get-secret-value --secret-id myapp/db/password

# 업데이트
aws secretsmanager update-secret --secret-id myapp/db/password \
  --secret-string '{"username":"app","password":"new"}'

# 삭제 (recovery window 7-30일)
aws secretsmanager delete-secret --secret-id myapp/db/password \
  --recovery-window-in-days 14
```

### 5.2 Terraform

```hcl
resource "aws_secretsmanager_secret" "db" {
  name        = "myapp/db/credentials"
  kms_key_id  = aws_kms_key.myapp.arn
  description = "myapp Postgres credentials"
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    username = "app"
    password = random_password.db.result
    host     = aws_db_instance.myapp.address
    port     = 5432
    dbname   = "app"
  })
}
```

### 5.3 SDK (Python)

```python
import boto3, json
sm = boto3.client("secretsmanager")

resp = sm.get_secret_value(SecretId="myapp/db/credentials")
creds = json.loads(resp["SecretString"])

conn = psycopg2.connect(
    host=creds["host"], port=creds["port"],
    dbname=creds["dbname"],
    user=creds["username"],
    password=creds["password"],
    sslmode="require",
)
```

→ 응용 시작 시 1회 fetch + memory cache. 또는 X 초마다 refresh.

### 5.4 ECS / Lambda 통합

```json
{
  "containerDefinitions": [{
    "name": "app",
    "secrets": [
      { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:...:secret:myapp/db/credentials:password::" }
    ]
  }]
}
```

→ ECS 가 task 시작 시 자동 환경변수 주입.

Lambda:
```hcl
environment {
  variables = {
    DB_SECRET_ARN = aws_secretsmanager_secret.db.arn
  }
}
```
응용이 fetch.

---

## 6. 자동 Rotation

### 6.1 RDS / DocumentDB / Redshift — native

```hcl
resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = aws_lambda_function.rotator.arn
  rotation_rules {
    automatically_after_days = 30
  }
}
```

AWS 가 Lambda template 제공 (multi-user / single-user).

### 6.2 자체 응용 — Lambda 작성

```python
def lambda_handler(event, context):
    step = event["Step"]               # createSecret / setSecret / testSecret / finishSecret
    if step == "createSecret":
        # 새 비밀 생성 → AWSPENDING
        ...
    elif step == "setSecret":
        # 외부 시스템 (DB user) 에 적용
        ...
    elif step == "testSecret":
        # 새 비밀로 연결 test
        ...
    elif step == "finishSecret":
        # AWSPENDING → AWSCURRENT
        ...
```

---

## 7. Cross-Region Replication

```hcl
resource "aws_secretsmanager_secret" "db" {
  name = "myapp/db"
  replica {
    region = "us-east-1"
  }
}
```

→ DR 시 다른 region 에서도 접근.

---

## 8. Resource Policy (Cross-account)

```hcl
resource "aws_secretsmanager_secret_policy" "shared" {
  secret_arn = aws_secretsmanager_secret.db.arn
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::OTHER_ACCOUNT:root" }
      Action    = "secretsmanager:GetSecretValue"
      Resource  = "*"
    }]
  })
}
```

---

## 9. 비용

```
$0.40 / secret·월
$0.05 / 10K API call
KMS 호출 별도 (envelope)
```

매월 ~ $0.5-$5 / secret. 큰 organization = Parameter Store 검토.

---

## 10. 사용 시나리오

- RDS / Aurora password (rotation)
- 외부 API token (Stripe / SendGrid)
- TLS private key (보통 ACM 사용)
- OAuth client secret
- service account credentials

---

## 11. 함정

### 11.1 secret 의 logging
Lambda / 응용이 secret 을 log 하면 leak. 마스킹.

### 11.2 rotation 의 응용 영향
rotation 직후 옛 secret = 무효. 응용이 캐시 너무 길면 fail. 짧은 cache TTL.

### 11.3 delete 의 recovery window
즉시 삭제 X. 7-30일. 의도적 삭제는 짧게.

### 11.4 KMS key 분실
KMS key 삭제 = secret 영구 손실. 키 보호.

### 11.5 cost
1000 secrets × $0.40 = $400/월. 작은 secret 은 Parameter Store.

### 11.6 cross-region replication 후 폐기
secondary region 별도 삭제.

### 11.7 plaintext logging in CloudTrail
CloudTrail = API call 만 (value X). 안전.

---

## 12. SSM Parameter Store — 대안

```bash
aws ssm put-parameter --name "/myapp/db/password" --value "..." --type SecureString --key-id alias/myapp
aws ssm get-parameter --name "/myapp/db/password" --with-decryption
```

```hcl
resource "aws_ssm_parameter" "db_password" {
  name  = "/myapp/db/password"
  type  = "SecureString"
  value = random_password.db.result
  key_id = aws_kms_key.myapp.arn
}
```

ECS:
```json
"secrets": [{ "name": "DB_PASSWORD", "valueFrom": "arn:aws:ssm:...parameter/myapp/db/password" }]
```

→ 거의 같은 인터페이스. 무료. rotation 만 직접.

---

## 13. 학습 자료

- AWS Secrets Manager docs
- **Rotating Secrets** AWS blog
- **External Secrets Operator** (K8s)
- **HashiCorp Vault**

---

## 14. 관련

- [[security]] — Security hub
- [[iam]] — Secret access
- [[kms]] — 암호화
- [[rds]] — RDS rotation
- [[lambda]] — rotation Lambda
