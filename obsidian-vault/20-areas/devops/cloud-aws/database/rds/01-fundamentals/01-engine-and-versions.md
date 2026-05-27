---
title: "RDS 01-1 — Engines + Versions (Postgres / MySQL / Oracle / SQL Server)"
kind: knowledge
status: current
created_at: 2026-05-22T19:52:00+09:00
tags: [aws, rds, fundamentals, engine, postgres, mysql, oracle, sql-server]
home_hub: rds
---

# RDS Engines + Versions

> 5 engine — Postgres / MySQL / MariaDB / Oracle / SQL Server. Standard Support → Extended Support → EOL. Major upgrade Blue-Green 권장.

## 1. Engine 별 특징

| Engine | License | 권장 |
| --- | --- | --- |
| **Postgres** | open source | ★ default |
| **MySQL** | open source (Oracle) | 일반 web |
| **MariaDB** | open source | MySQL fork |
| **Oracle** | BYOL / License Included | enterprise legacy |
| **SQL Server** | License Included | Microsoft stack |

## 2. Postgres 버전 정책

| Version | Released | Standard Support EOL | Extended Support EOL |
| --- | --- | --- | --- |
| 16 | 2023-09 | 2027-11 | 2030-11 |
| 15 | 2022-10 | 2027-02 | 2030-02 |
| 14 | 2021-09 | 2026-11 | 2029-11 |
| 13 | 2020-09 | 2026-02 | 2029-02 |
| 12 | 2019-10 | EOL | 2027-02 |
| 11 | 2018-10 | EOL | 2026-02 |

> Standard Support 만료 후 = Extended Support ($$$$, $0.10/vCPU·hour).

## 3. Extended Support 비용

```
db.t4g.medium (2 vCPU):
  Standard:           $0.103/h
  + Extended Support: $0.20/h
  = $0.303/h (~3x)
```

> 적시 upgrade 권장.

## 4. Engine 생성

```bash
aws rds create-db-instance \
  --db-instance-identifier bkurs-prod \
  --engine postgres \
  --engine-version 16.2 \
  --db-instance-class db.t4g.medium \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted --kms-key-id alias/bkurs-rds \
  --master-username bkurs_admin \
  --manage-master-user-password \
  --master-user-secret-kms-key-id alias/secrets \
  --db-name bkurs \
  --multi-az \
  --backup-retention-period 35 \
  --preferred-backup-window "18:00-19:00" \
  --preferred-maintenance-window "Sun:19:00-Sun:20:00" \
  --enable-iam-database-authentication \
  --enable-performance-insights --performance-insights-retention-period 7 \
  --monitoring-interval 60 --monitoring-role-arn arn:aws:iam::ACCT:role/rds-monitoring \
  --enable-cloudwatch-logs-exports postgresql upgrade \
  --deletion-protection \
  --vpc-security-group-ids sg-rds \
  --db-subnet-group-name bkurs-db-subnet \
  --tags Key=Environment,Value=prod
```

## 5. Engine version upgrade

### Minor (patch)
```bash
# Auto minor — preferred maintenance window 자동
# 또는 명시
aws rds modify-db-instance \
  --db-instance-identifier bkurs-prod \
  --engine-version 16.4 \
  --apply-immediately
```

### Major (16 → 17)
```bash
# Blue-Green 권장 (다른 doc)
# 또는 직접 upgrade (downtime)
aws rds modify-db-instance \
  --db-instance-identifier bkurs-prod \
  --engine-version 17.0 \
  --allow-major-version-upgrade \
  --apply-immediately
```

> Major 는 **앱 호환성 검증 필수**. Blue-Green 으로 zero-downtime ([[../02-operations/03-migration-and-blue-green]]).

## 6. Parameter Group

```bash
# Custom Parameter Group
aws rds create-db-parameter-group \
  --db-parameter-group-name bkurs-pg16 \
  --db-parameter-group-family postgres16 \
  --description "bkurs custom"

# Modify parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name bkurs-pg16 \
  --parameters "ParameterName=max_connections,ParameterValue=200,ApplyMethod=pending-reboot" \
               "ParameterName=shared_buffers,ParameterValue={DBInstanceClassMemory*1/4},ApplyMethod=pending-reboot" \
               "ParameterName=work_mem,ParameterValue=4096,ApplyMethod=immediate" \
               "ParameterName=log_statement,ParameterValue=mod,ApplyMethod=immediate" \
               "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate"

# Attach to instance
aws rds modify-db-instance \
  --db-instance-identifier bkurs-prod \
  --db-parameter-group-name bkurs-pg16 \
  --apply-immediately
```

| Parameter Type | 즉시 적용 | reboot 필요 |
| --- | --- | --- |
| dynamic | ✓ (`ApplyMethod=immediate`) | ✗ |
| static | ✗ | `ApplyMethod=pending-reboot` |

## 7. Postgres 권장 parameter

```
shared_buffers:               25% of RAM
work_mem:                     4MB-16MB
maintenance_work_mem:         1GB+
effective_cache_size:         50-75% of RAM
random_page_cost:             1.1 (SSD)
log_min_duration_statement:   1000ms (slow query)
log_statement:                "mod" (write 만)
pgaudit.log:                  "write,ddl"  (감사)
ssl:                          on
rds.force_ssl:                1  (TLS 강제)
```

## 8. Option Group (Oracle / SQL Server)

```bash
aws rds create-option-group \
  --option-group-name bkurs-oracle-options \
  --engine-name oracle-ee \
  --major-engine-version 19 \
  --option-group-description "Oracle TDE + APEX"

aws rds add-option-to-option-group --option-group-name bkurs-oracle-options \
  --options "OptionName=Oracle_TDE,OptionSettings=[{Name=ENCRYPTION_KEY,Value=...}]" "OptionName=APEX,OptionVersion=20.2.v1"
```

활용:
- Oracle TDE (encryption)
- Oracle SQLNET / OEM
- SQL Server Audit / TDE / MSDTC
- MySQL MEMCACHED

## 9. Engine 별 storage 한도

| | min | max |
| --- | --- | --- |
| Postgres / MySQL / MariaDB | 20 GB | 64 TiB (gp3) / 64 TiB (io2) |
| Oracle | 20 GB | 64 TiB |
| SQL Server (Express) | 20 GB | 16 TiB |
| SQL Server (Standard/Enterprise) | 20 GB | 16 TiB |

## 10. 다른 engine 추천 case

| Use Case | Engine |
| --- | --- |
| 일반 web/API | Postgres |
| WordPress / Drupal | MySQL / MariaDB |
| Magento / e-commerce | MySQL |
| .NET / Microsoft stack | SQL Server |
| Legacy enterprise | Oracle |
| Geographic / GIS | Postgres (PostGIS) |
| JSON-heavy | Postgres (JSONB) |
| Vector / AI | Postgres (pgvector) |

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Engine version EOL → Extended cost | 적시 upgrade |
| Parameter Group default 사용 | Custom (max_connections / log 등) |
| Static parameter 변경 후 reboot 안 함 | apply-immediately + reboot |
| MariaDB 와 MySQL 자유 전환 가정 | 같은 family 만 |
| Oracle BYOL vs LI 혼동 | License 확인 |
| Major upgrade 직접 → app 깨짐 | Blue-Green |

## 12. 다음

- ▶ [[02-instance-classes-and-pricing]]
- ▶ [[03-multi-az-and-read-replica]]
- ▶ [[../02-operations/03-migration-and-blue-green]]

## 13. 영상 / 자료

| 자료 |
| --- |
| [RDS Engines](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_DBEngine.html) |
| [Extended Support](https://aws.amazon.com/blogs/aws/announcing-amazon-rds-extended-support/) |

## 14. 참고

- [[../rds]]
- [[../../aurora/aurora]]
