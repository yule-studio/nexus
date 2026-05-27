---
title: "Aurora 01-2 — Engines + Versions (Postgres / MySQL / Babelfish / Limitless)"
kind: knowledge
status: current
created_at: 2026-05-22T20:24:00+09:00
tags: [aws, aurora, engine, postgres, mysql, babelfish, limitless]
home_hub: aurora
---

# Aurora Engines + Versions

> 2 engine — Aurora Postgres (Postgres compatible) / Aurora MySQL (MySQL compatible). Babelfish (SQL Server compatible). Limitless Database (Postgres 분산).

## 1. Aurora Engines

| | Aurora Postgres | Aurora MySQL | Babelfish | Limitless |
| --- | --- | --- | --- | --- |
| Compatibility | Postgres 12-16 | MySQL 5.7-8.0 | SQL Server T-SQL (Aurora MySQL 위) | Postgres (sharded) |
| Performance vs OSS | 3x | 5x | — | scale-out unlimited |
| Storage | Aurora distributed | Aurora distributed | Aurora MySQL | distributed shards |
| Serverless v2 | ✓ | ✓ | ✓ | ✗ |
| Global Database | ✓ | ✓ | ✓ | ✗ |
| Backtrack | ✗ | ✓ (72h) | ✓ | ✗ |
| Use | 일반 default | high RPS | SQL Server 호환 | petabyte scale |

## 2. Aurora Postgres Versions

| Version | Released | Standard Support EOL | Extended Support EOL |
| --- | --- | --- | --- |
| 16 | 2023-12 | 2027-11 | 2030-11 |
| 15 | 2022-12 | 2027-02 | 2030-02 |
| 14 | 2022-04 | 2026-11 | 2029-11 |
| 13 | 2021-09 | 2026-02 | 2029-02 |
| 12 | 2020-09 | EOL | 2027-02 |

> RDS Postgres 와 비슷한 cycle. Extended Support 비용 동일 ($0.10/vCPU·hour).

## 3. Aurora MySQL Versions

| Version | Compatibility | Notes |
| --- | --- | --- |
| Aurora MySQL 3 | MySQL 8.0 | 권장 ★ |
| Aurora MySQL 2 | MySQL 5.7 | EOL 2024-10 |
| Aurora MySQL 1 | MySQL 5.6 | EOL |

## 4. Cluster 생성

```bash
aws rds create-db-cluster \
  --db-cluster-identifier bkurs-aurora \
  --engine aurora-postgresql \
  --engine-version 16.2 \
  --master-username bkurs_admin \
  --manage-master-user-password \
  --database-name bkurs \
  --vpc-security-group-ids sg-rds \
  --db-subnet-group-name bkurs-db-subnet \
  --storage-encrypted --kms-key-id alias/bkurs-aurora \
  --backup-retention-period 35 \
  --preferred-backup-window "18:00-19:00" \
  --enable-iam-database-authentication \
  --enable-cloudwatch-logs-exports postgresql \
  --deletion-protection \
  --tags Key=Environment,Value=prod

# Instance 생성 (Writer)
aws rds create-db-instance \
  --db-instance-identifier bkurs-aurora-writer \
  --db-cluster-identifier bkurs-aurora \
  --engine aurora-postgresql \
  --db-instance-class db.r6g.large \
  --enable-performance-insights

# Reader 추가
aws rds create-db-instance \
  --db-instance-identifier bkurs-aurora-reader-1 \
  --db-cluster-identifier bkurs-aurora \
  --engine aurora-postgresql \
  --db-instance-class db.r6g.large \
  --availability-zone ap-northeast-2c
```

## 5. Parameter Group (Cluster + Instance)

Aurora 는 **2 종 parameter group**:

### Cluster Parameter Group
storage / replication / global parameter.
```bash
aws rds create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name bkurs-aurora-pg16-cluster \
  --db-parameter-group-family aurora-postgresql16 \
  --description "cluster level"

aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name bkurs-aurora-pg16-cluster \
  --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate"
```

### Instance Parameter Group
instance level (max_connections, shared_buffers, etc).
```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name bkurs-aurora-pg16 \
  --db-parameter-group-family aurora-postgresql16 \
  --description "instance level"
```

## 6. Babelfish (SQL Server 호환)

```bash
# Aurora MySQL 8.0 위 활성
aws rds create-db-cluster \
  --engine aurora-postgresql \
  --engine-version 14.6 \
  --enable-babelfish \
  --master-username sa \
  ...

# TDS endpoint port 1433
aws rds describe-db-clusters --db-cluster-identifier bkurs-babelfish \
  --query 'DBClusters[0].Endpoints'
```

→ T-SQL / SQL Server protocol (TDS) 호환. SQL Server → Aurora migration.

## 7. Aurora Postgres extension

Standard Postgres extensions:
- `pgaudit` — audit log
- `pgcrypto` — encryption
- `pgvector` — vector / embedding
- `postgis` — geographic
- `pg_partman` — partition
- `pg_stat_statements` — query analysis
- `tds_fdw` — SQL Server FDW
- `aws_s3` — S3 import/export (Aurora 만)
- `aws_lambda` — Lambda invoke (Aurora 만)
- `aws_commons` — utilities

```sql
-- Install
CREATE EXTENSION IF NOT EXISTS pgaudit;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pgvector;
```

## 8. Aurora MySQL extension

- MySQL plugin
- Aurora Backtrack
- Aurora ML (SageMaker / Comprehend 직접 호출)
- Aurora MySQL parallel query
- Aurora Replicas (binlog 활용한 외부 replica 가능)

## 9. Aurora Postgres vs RDS Postgres

| | Aurora Postgres | RDS Postgres |
| --- | --- | --- |
| Storage | distributed (~3x) | EBS (gp3/io2) |
| Read Replica | 15 max, < 100ms | 5 max, async |
| Failover | ~30초 | 1-2분 |
| Backup | continuous (PITR 1초) | daily + 5분 Tx log (PITR 1분) |
| Global DB | ✓ | ✗ (manual replica) |
| Serverless | ✓ (v2) | ✗ |
| Backtrack | ✗ (PITR 활용) | ✗ |
| Cost | premium ~20% | baseline |
| Extension | `aws_s3`, `aws_lambda`, Babelfish | 표준 만 |

## 10. Engine version upgrade

### Minor
```bash
aws rds modify-db-cluster --db-cluster-identifier bkurs-aurora \
  --engine-version 16.4 --apply-immediately
```

### Major
```bash
# Aurora 의 Blue-Green Deployment (2022+)
aws rds create-blue-green-deployment \
  --blue-green-deployment-name bkurs-upgrade \
  --source arn:aws:rds:...:cluster:bkurs-aurora \
  --target-engine-version 17.0
```

## 11. Aurora Custom Endpoint

```bash
aws rds create-db-cluster-endpoint \
  --db-cluster-identifier bkurs-aurora \
  --db-cluster-endpoint-identifier bkurs-analytics \
  --endpoint-type READER \
  --static-members bkurs-aurora-reader-3 bkurs-aurora-reader-4 \
  --tags Key=Purpose,Value=analytics
```

→ analytics workload 만 별도 Reader 사용.

## 12. Limitless Database (Aurora Postgres, 2024+)

```
Petabyte scale Postgres:
- Sharded automatic (router 가 routing)
- Single endpoint
- Standard SQL (cross-shard query)
- Production 단일 cluster — millions TPS
```

> 일반 Aurora 가 부족할 정도의 scale 만.

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Aurora MySQL 2 (EOL) | v3 |
| Cluster vs Instance parameter group 혼동 | 둘 다 별도 |
| Extension install (DB 안에서) | CREATE EXTENSION |
| Aurora Postgres 에서 Backtrack | MySQL only — PITR |
| Babelfish + SQL Server 100% 호환 가정 | SCT 의 compatibility report |
| Custom Endpoint 의 static member | 자동 추가 안 됨 (TYPE=READER) |

## 14. 다음

- ▶ [[03-serverless-v2]]
- ▶ [[../02-operations/01-cluster-and-endpoint]]
- ▶ [[../02-operations/02-global-database-and-dr]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Aurora Postgres](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraPostgreSQL.html) |
| [Babelfish](https://aws.amazon.com/rds/aurora/babelfish/) |
| [Limitless Database](https://aws.amazon.com/rds/aurora/limitless/) |

## 16. 참고

- [[../aurora]]
- [[01-architecture-deep]]
