---
title: "AWS RDS — Relational Database Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:40:00+09:00
tags:
  - aws
  - database
  - rds
---

# AWS RDS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RDS 개념 + 사용 |

**[[database|↑ Database]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**매니지드 RDB** — PostgreSQL / MySQL / MariaDB / Oracle / SQL Server.
backup / patch / Multi-AZ / read replica 자동.

---

## 2. 왜

- self-managed (EC2 + Postgres) = patch / backup / failover 운영 부담
- RDS = 클릭 한 번에 enterprise-grade
- 단점: 비용 (~ 2x), 일부 OS 권한 X, 옛 버전 종료

대안:
- **Aurora** — AWS 자체 — 더 빠름 + 비싸지만 분산
- **DynamoDB** — NoSQL
- **self-managed on EC2** — 100% 제어 + 운영 부담
- **외부 매니지드** — Supabase / Neon / PlanetScale

---

## 3. 엔진

| Engine | 의미 |
| --- | --- |
| **PostgreSQL** | 가장 권장 (확장 + JSONB + PostGIS + pgvector) |
| **MySQL** | 표준 |
| **MariaDB** | MySQL fork |
| **Oracle** | 라이선스 (BYOL / License Included) |
| **SQL Server** | 라이선스 |
| **Db2** | 옛 IBM |

DB 자체의 운영 = [[database|↗ 20-areas/database]] 참조.
여기는 **AWS 측면** (instance / parameter / backup / network / 비용).

---

## 4. 설치 / 첫 인스턴스

### 4.1 Console
RDS → Create database → Standard create → PostgreSQL 16.

### 4.2 CLI

```bash
aws rds create-db-instance \
  --db-instance-identifier myapp-prod \
  --engine postgres \
  --engine-version 16.3 \
  --db-instance-class db.t3.medium \
  --allocated-storage 100 \
  --storage-type gp3 \
  --master-username postgres \
  --master-user-password '...' \
  --vpc-security-group-ids sg-xxx \
  --db-subnet-group-name my-subnet-group \
  --backup-retention-period 7 \
  --preferred-backup-window "16:00-17:00" \
  --preferred-maintenance-window "Sun:18:00-Sun:20:00" \
  --multi-az \
  --storage-encrypted \
  --kms-key-id alias/aws/rds \
  --deletion-protection
```

### 4.3 Terraform

```hcl
resource "aws_db_instance" "myapp" {
  identifier                  = "myapp-prod"
  engine                      = "postgres"
  engine_version              = "16.3"
  instance_class              = "db.t3.medium"
  allocated_storage           = 100
  max_allocated_storage       = 1000              # autoscale
  storage_type                = "gp3"
  storage_encrypted           = true
  kms_key_id                  = aws_kms_key.rds.arn

  db_name                     = "app"
  username                    = "postgres"
  password                    = var.db_password
  port                        = 5432

  vpc_security_group_ids      = [aws_security_group.rds.id]
  db_subnet_group_name        = aws_db_subnet_group.private.name
  publicly_accessible         = false

  multi_az                    = true
  backup_retention_period     = 7
  backup_window               = "16:00-17:00"
  maintenance_window          = "Sun:18:00-Sun:20:00"

  performance_insights_enabled = true
  enabled_cloudwatch_logs_exports = ["postgresql"]

  deletion_protection         = true
  skip_final_snapshot         = false
  final_snapshot_identifier   = "myapp-prod-final"
}
```

---

## 5. Multi-AZ

```
Primary (AZ-A)
   ↓ synchronous
Standby (AZ-B)        — primary 죽으면 자동 failover (1-2 분)
```

- **Single-AZ** ~ $60/월 (db.t3.medium pg)
- **Multi-AZ** ~ $120/월 (2배)

추가:
- **Multi-AZ DB cluster** (3 인스턴스, 2 readable standby) — 더 빠른 failover (~ 35초)

---

## 6. Read Replica

```
Primary (write)
  ↓ async replication
Replica (read-only, 같은/다른 region)
```

- 최대 15 replicas
- cross-region 가능 (DR)
- read 부하 분산
- replica 를 promote → 독립 primary

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-replica \
  --source-db-instance-identifier myapp-prod \
  --availability-zone ap-northeast-2c
```

---

## 7. Backup

| 종류 | 동작 |
| --- | --- |
| **Automated backup** | 매일 + transaction log → PITR (Point-in-Time Recovery) |
| **Manual snapshot** | 보존 — 사용자 명시 삭제까지 |
| **Cross-region snapshot** | 다른 region 복사 |
| **AWS Backup** | 중앙 정책 |

```bash
# 수동
aws rds create-db-snapshot --db-instance-identifier myapp --db-snapshot-identifier myapp-2026-05-14

# 복원 — 새 인스턴스
aws rds restore-db-instance-from-db-snapshot --db-instance-identifier myapp-restored --db-snapshot-identifier myapp-2026-05-14

# PITR — 특정 시각
aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier myapp --target-db-instance-identifier myapp-pitr --restore-time 2026-05-14T12:00:00Z
```

→ retention 7-35일. cross-region backup 권장.

---

## 8. Parameter / Option Group

| | 의미 |
| --- | --- |
| **Parameter Group** | `postgresql.conf` 의 일부 (work_mem, shared_buffers, max_connections, ...) |
| **Option Group** | 추가 기능 (TDE, log export 등) |

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name myapp-pg16 \
  --db-parameter-group-family postgres16 \
  --description "myapp pg16"

aws rds modify-db-parameter-group \
  --db-parameter-group-name myapp-pg16 \
  --parameters "ParameterName=work_mem,ParameterValue=32MB,ApplyMethod=immediate"
```

→ 일부는 reboot 필요 (`ApplyMethod=pending-reboot`).

`shared_buffers` / `max_connections` 등 자세히 → [[configuration]]

---

## 9. 접속

```python
import psycopg2
conn = psycopg2.connect(
    host="myapp-prod.xxx.ap-northeast-2.rds.amazonaws.com",
    port=5432,
    dbname="app",
    user="postgres",
    password=os.environ["DB_PASSWORD"],
    sslmode="require",                       # TLS
)
```

→ password 는 **Secrets Manager** 또는 IAM auth.
자세히 → [[secrets-manager]]

### 9.1 IAM Authentication

```bash
# 토큰 (15분 유효)
TOKEN=$(aws rds generate-db-auth-token --hostname xxx --port 5432 --username myuser)
psql "host=xxx port=5432 dbname=app user=myuser sslmode=require" password=$TOKEN
```

→ 자격증명 storage 없음. IAM role 만.

---

## 10. Performance Insights

```
RDS Console → Performance Insights
- 어떤 SQL 이 wait time 차지
- 어떤 wait event (CPU / Lock / IO)
- 시간대 별 부하
```

자체 모니터링 = pg_stat_statements + CloudWatch. 자세히 → [[cloudwatch]]

---

## 11. 보안

- **VPC 안 private subnet** — public X
- **Security Group** — 응용 SG 만 5432 허용
- **TLS** 강제 (parameter group)
- **암호화** at rest (KMS) + transit (TLS)
- **IAM auth** 또는 Secrets Manager (자동 rotation)

자세히 → [[secrets-manager]], [[vpc]]

---

## 12. 비용 (Seoul)

| 항목 | $ |
| --- | --- |
| db.t3.medium pg single-AZ | ~ $60/월 |
| db.t3.medium pg multi-AZ | ~ $120/월 |
| db.r6g.large pg multi-AZ | ~ $400/월 |
| storage gp3 100 GB | ~ $15/월 |
| backup storage (over allocated) | $0.095/GB |
| data transfer cross-AZ | $0.01/GB |

→ multi-AZ + 큰 인스턴스 + read replica = 빠르게 $1000+/월.

---

## 13. 사용 시나리오

- 일반 OLTP 응용 (대부분)
- 작은 ~ 중간 SaaS
- 분석 (작은 — 큰 건 Redshift)

부적합:
- 초저 latency (자체 EC2)
- 거대 TPS (Aurora / Cassandra)
- multi-master (Aurora multi-master, 옛)
- 매니지드 안 되는 extension

---

## 14. 함정

### 14.1 storage autoscaling 없음 + 가득
공간 부족 → 응용 fail. `max_allocated_storage` 설정.

### 14.2 maintenance window
업그레이드 / patch 시 짧은 다운타임. 야간 / 트래픽 적은 시각.

### 14.3 deletion protection 끄기 잊음
실수 삭제. 운영 인스턴스 항상 ON.

### 14.4 publicly_accessible=true
private 만. 잘못 켜면 인터넷 노출 (SG 가 막아도 위험).

### 14.5 max_connections 한계
인스턴스 class 비례. PgBouncer / RDS Proxy 권장.

### 14.6 RDS Proxy
connection pool — Lambda / 큰 동시성에 유용.

### 14.7 minor version 자동 업그레이드
권장하지만 회귀 위험 — staging 먼저.

### 14.8 옛 engine 종료
extended support 비용 ↑. 정기 마이그.

---

## 15. RDS Proxy

```hcl
resource "aws_db_proxy" "myapp" {
  name                   = "myapp-proxy"
  engine_family          = "POSTGRESQL"
  role_arn               = aws_iam_role.proxy.arn
  vpc_subnet_ids         = aws_subnet.private[*].id
  require_tls            = true
  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "DISABLED"
    secret_arn  = aws_secretsmanager_secret.db.arn
  }
}
```

→ 응용은 proxy 에 connect. RDS Proxy 가 풀 / failover 처리.

---

## 16. 학습 자료

- AWS RDS docs
- AWS Database Blog
- **PostgreSQL on AWS** whitepaper

---

## 17. 관련

- [[database]] — Database hub
- [[aurora]] — 분산 RDB
- [[dynamodb]] — NoSQL
- [[postgresql|↗ Postgres 자체]]
- [[secrets-manager]]
- [[vpc]]
