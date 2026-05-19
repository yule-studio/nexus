---
title: "AWS Aurora — Cloud-native RDB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:45:00+09:00
tags:
  - aws
  - database
  - aurora
---

# AWS Aurora

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Aurora 개념 + 사용 |

**[[database|↑ Database]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 자체 설계의 **cloud-native RDB**. PostgreSQL / MySQL **wire-compatible** + 분산 storage (6 copy across 3 AZ) + 빠른 failover + read scale.

---

## 2. 왜

- **RDS Postgres / MySQL** = 일반 OS 위의 single instance
- Aurora = AWS 가 분산 storage 새로 설계 → **3-5x 빠름 (claims)**, 더 빠른 failover, online resize
- BUT 더 비쌈 (~1.2-2x RDS)
- vendor lock (Postgres wire compatible 이지만 일부 함정)

대안:
- **RDS Postgres / MySQL** — 단순 / 이식성
- **DynamoDB** — NoSQL
- **CockroachDB / YugabyteDB** — multi-cloud 분산 RDB

---

## 3. 아키텍처

```
응용
  ↓
Aurora Writer (1 인스턴스)
  ↓ via storage protocol (not WAL replication)
[Distributed Storage]
  ↓ ↓ ↓ ↓ ↓ ↓
6 copies × 3 AZ
  ↑
Aurora Readers (최대 15)
```

특징:
- storage 가 자동 6-way replicate
- writer 죽으면 reader 1 명 promote (수십 초)
- storage auto-scale (10 GB → 128 TB)
- crash recovery 매우 빠름 (storage 가 일관성 보장)

---

## 4. Aurora vs RDS

| | RDS | Aurora |
| --- | --- | --- |
| 엔진 | 표준 PG/MySQL/... | PG/MySQL 호환 (자체) |
| 가격 | 기본 | ~1.2-2x |
| Replica | 5 (lag 있음) | 15 (저 lag) |
| Failover | 1-2분 | < 30초 |
| Storage | 미리 할당 | 사용한 만큼 자동 |
| Backup | snapshot | continuous |
| Multi-master | X (multi-AZ standby) | 있음 (옛) |
| Global database | X | 가능 (cross-region) |

---

## 5. 설치 / 첫 cluster

### 5.1 Console
RDS → Create database → Aurora (PostgreSQL Compatible).

### 5.2 CLI

```bash
# cluster
aws rds create-db-cluster \
  --db-cluster-identifier myapp-aurora \
  --engine aurora-postgresql \
  --engine-version 16.3 \
  --master-username postgres \
  --master-user-password '...' \
  --db-subnet-group-name my-subnet-group \
  --vpc-security-group-ids sg-xxx \
  --backup-retention-period 7 \
  --storage-encrypted \
  --kms-key-id alias/aws/rds

# 인스턴스 (writer + readers)
aws rds create-db-instance \
  --db-instance-identifier myapp-aurora-writer \
  --db-cluster-identifier myapp-aurora \
  --engine aurora-postgresql \
  --db-instance-class db.r6g.large

aws rds create-db-instance \
  --db-instance-identifier myapp-aurora-reader-1 \
  --db-cluster-identifier myapp-aurora \
  --engine aurora-postgresql \
  --db-instance-class db.r6g.large
```

### 5.3 Terraform

```hcl
resource "aws_rds_cluster" "myapp" {
  cluster_identifier   = "myapp-aurora"
  engine               = "aurora-postgresql"
  engine_version       = "16.3"
  database_name        = "app"
  master_username      = "postgres"
  master_password      = var.password
  backup_retention_period = 7
  storage_encrypted    = true

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name

  serverlessv2_scaling_configuration {
    min_capacity = 0.5
    max_capacity = 4.0
  }
}

resource "aws_rds_cluster_instance" "writer" {
  cluster_identifier = aws_rds_cluster.myapp.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.myapp.engine
}
```

---

## 6. Endpoint

```
Writer endpoint: myapp-aurora.cluster-xxx.ap-northeast-2.rds.amazonaws.com
Reader endpoint: myapp-aurora.cluster-ro-xxx....                            (자동 LB)
Custom endpoint: myapp-aurora.cluster-custom-...                            (subset)
Instance endpoint: 각 인스턴스 직접
```

→ 응용:
- write = writer endpoint
- read = reader endpoint (자동 round-robin)

---

## 7. Aurora Serverless v2

```
용량 = Aurora Capacity Unit (ACU) — 0.5 ~ 256
자동 scale up/down (초 단위)
사용한 ACU·시간만 과금
```

```hcl
serverlessv2_scaling_configuration {
  min_capacity = 0.5
  max_capacity = 16.0
}
```

→ 변동 부하 / dev / 가끔 사용. 항상 부하면 provisioned 가 더 쌈.

---

## 8. Global Database

```
Primary region (write + read)
  ↓ < 1초 lag, dedicated infrastructure
Secondary region (read-only, optional write promote)
```

- DR (region 전체 장애)
- low-latency global read
- < 1분 promote

---

## 9. 비용 (Seoul)

| 항목 | $ |
| --- | --- |
| db.r6g.large pg | ~ $370/월 |
| db.r6g.xlarge | ~ $740/월 |
| Serverless v2 ACU·시간 | $0.16 |
| Storage | $0.10/GB·월 (사용한 만큼) |
| I/O | $0.20/M (Standard) |
| Backup | $0.021/GB·월 (over 100%) |
| Global Database — secondary region | + |

Aurora I/O-Optimized — I/O 무료 + storage 더 비쌈. 큰 IOPS 워크로드 = optimized.

---

## 10. 사용 시나리오

- 운영 OLTP — RDS 보다 빠른 / 큰 규모
- read-heavy + 많은 replica
- global multi-region
- variable / spiky (Serverless v2)

부적합:
- 작은 / 단순 (RDS 가 충분)
- 옛 Postgres / MySQL feature 의존
- multi-cloud 이식

---

## 11. PostgreSQL / MySQL 호환의 함정

- 일부 extension 미지원 / 옛 버전
- `pg_dump` / `mysqldump` 는 호환
- 일부 system catalog 다름
- Aurora-specific 기능 (faster index, parallel query) 활용 시 lock-in

→ Postgres-on-RDS 와 호환되도록 작성하면 마이그 자유.

---

## 12. Aurora Cloning

```bash
# 즉시 clone (copy-on-write, 거의 비용 0)
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier myapp-aurora \
  --db-cluster-identifier myapp-clone \
  --restore-type copy-on-write \
  --use-latest-restorable-time
```

→ staging 환경 production 데이터 즉시 복제. 변경만 storage 추가.

---

## 13. Backtrack (MySQL 만)

```
실수 후 N 시간 전으로 즉시 되감기 (storage 가 versioned)
```

`backtrack-window` 활성 후. PostgreSQL 은 X.

---

## 14. 함정

### 14.1 Aurora I/O 비용 폭증
I/O-heavy 워크로드 → optimized 또는 캐시.

### 14.2 Serverless v2 의 cold start
0 ACU 까진 안 됨 (최소 0.5). 진짜 0 → idle pause (v2 옵션).

### 14.3 Multi-master (옛 MySQL)
deprecated. 일반 multi-AZ + reader.

### 14.4 Global Database 의 secondary
write 비싸고 데이터 잃을 수 있음 (promotion).

### 14.5 Aurora 의 fast clone
같은 region 만. 빠르지만 cross-region 은 snapshot 으로.

---

## 15. 학습 자료

- AWS Aurora docs
- **Aurora 백서** — DSDB Paper (Verbitski et al., 2017)
- **AWS re:Invent** Aurora deep dive

---

## 16. 관련

- [[database]] — DB hub
- [[rds]] — 일반 RDS
- [[dynamodb]] — NoSQL
- [[postgresql|↗ Postgres 자체]]
