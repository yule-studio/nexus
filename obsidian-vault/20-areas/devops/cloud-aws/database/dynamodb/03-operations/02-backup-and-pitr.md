---
title: "DynamoDB 03-2 — PITR + On-Demand Backup + AWS Backup"
kind: knowledge
status: current
created_at: 2026-05-22T21:08:00+09:00
tags: [aws, dynamodb, backup, pitr, restore, aws-backup, dr]
home_hub: dynamodb
---

# DynamoDB Backup + PITR

> Continuous Point-In-Time Recovery (35d, 1-second granularity) + On-Demand Backup (영구) + AWS Backup 통합.

## 1. 2 종류

| | PITR | On-Demand Backup |
| --- | --- | --- |
| Trigger | 자동 continuous | 사용자 명시 |
| Retention | 35d | 영구 |
| Granularity | 1 second | snapshot 시점 |
| Cost | $0.20/GB·월 (continuous) | $0.10/GB·월 (storage) |
| Cross-region copy | ✓ (AWS Backup) | ✓ (AWS Backup) |
| Restore | 새 table | 새 table |

## 2. PITR 활성

```bash
aws dynamodb update-continuous-backups --table-name bkurs-orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# 상태 확인
aws dynamodb describe-continuous-backups --table-name bkurs-orders \
  --query 'ContinuousBackupsDescription'
```

## 3. PITR Restore

```bash
# 가능한 시간 범위
aws dynamodb describe-continuous-backups --table-name bkurs-orders \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription'

# Restore to specific time (새 table)
aws dynamodb restore-table-to-point-in-time \
  --source-table-name bkurs-orders \
  --target-table-name bkurs-orders-restored \
  --restore-date-time "2026-05-22T15:30:00Z" \
  --billing-mode-override PAY_PER_REQUEST \
  --sse-specification-override Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/bkurs-ddb

aws dynamodb wait table-exists --table-name bkurs-orders-restored
```

| 옵션 | 의미 |
| --- | --- |
| `--use-latest-restorable-time` | 가장 최근 (보통 ~수분 전) |
| `--restore-date-time` | 명시 시간 |
| `--billing-mode-override` | 새 mode 가능 |
| `--global-secondary-index-override` | GSI 변경 가능 |
| `--sse-specification-override` | KMS key 변경 |

## 4. On-Demand Backup

```bash
# Backup 생성
aws dynamodb create-backup --table-name bkurs-orders \
  --backup-name bkurs-orders-pre-migration

# List
aws dynamodb list-backups --table-name bkurs-orders

# Restore
aws dynamodb restore-table-from-backup \
  --target-table-name bkurs-orders-restored \
  --backup-arn arn:aws:dynamodb:ap-northeast-2:ACCT:table/bkurs-orders/backup/01716365400000-abcdef
```

## 5. AWS Backup 통합

```bash
# Backup Plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "bkurs-ddb-daily",
  "Rules": [{
    "RuleName": "daily",
    "TargetBackupVaultName": "bkurs-vault-aps2",
    "ScheduleExpression": "cron(0 18 * * ? *)",
    "Lifecycle": {
      "MoveToColdStorageAfterDays": 30,
      "DeleteAfterDays": 365
    },
    "CopyActions": [{
      "DestinationBackupVaultArn": "arn:aws:backup:ap-northeast-1:ACCT:backup-vault:bkurs-vault-apn1",
      "Lifecycle": { "DeleteAfterDays": 365 }
    }]
  }]
}'

# Selection (tag-based)
aws backup create-backup-selection --backup-plan-id $PLAN_ID \
  --backup-selection '{
    "SelectionName": "ddb-prod",
    "IamRoleArn": "arn:aws:iam::ACCT:role/service-role/AWSBackupDefaultServiceRole",
    "ListOfTags": [{
      "ConditionType": "STRINGEQUALS",
      "ConditionKey": "Environment",
      "ConditionValue": "prod"
    }]
  }'
```

## 6. Cross-region Backup Copy

```bash
# Manual
aws dynamodb create-backup --table-name bkurs-orders --backup-name dr-backup-$(date +%Y%m%d)

# AWS Backup 이 자동 cross-region copy
```

> AWS Backup 의 CopyActions 가 cross-region DR 표준.

## 7. Restore 결과

```
새 table:
- Items: source 의 backup 시점 그대로
- GSI: source 와 동일 (override 가능)
- Stream: 비활성 (별도 enable)
- Capacity mode: source 와 동일 (override 가능)
- Auto Scaling: 비활성 (재설정 필요)
- IAM permission: 별도 설정 필요
- Tag: source 의 tag 일부 (Backup 의 tag 추가)
```

## 8. Restore 시간

```
Table size  | Restore time
< 1 GB      | ~수분
10 GB       | ~10분
100 GB      | ~30분
1 TB        | ~수시간
```

→ 큰 table 의 DR = 시간 여유 필요.

## 9. Backup vs Streams + S3 export

| | Backup | S3 Export |
| --- | --- | --- |
| Format | DDB internal | Parquet / DynamoDB JSON |
| Restore | new DDB table | Athena query |
| Cost | $0.10/GB | $0.10/GB (export) + S3 |
| 권장 | DR / accidental delete | analytics / ML |

### S3 Export
```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:ap-northeast-2:ACCT:table/bkurs-orders \
  --s3-bucket bkurs-export \
  --export-time "2026-05-22T15:00:00Z" \
  --export-format DYNAMODB_JSON

# 또는 Parquet (Athena query)
--export-format ION
```

→ Athena 로 SQL query 가능.

## 10. S3 Import (Restore from S3)

```bash
aws dynamodb import-table \
  --s3-bucket-source S3Bucket=bkurs-export,S3KeyPrefix=AWSDynamoDB/exports/... \
  --input-format DYNAMODB_JSON \
  --table-creation-parameters '{
    "TableName": "bkurs-orders-restored",
    "AttributeDefinitions": [...],
    "KeySchema": [...],
    "BillingMode": "PAY_PER_REQUEST"
  }'
```

> S3 → DDB import. Cross-region migration 도 가능.

## 11. Backup 의 보안

```
PITR backup: same KMS key as source (default)
On-Demand backup: source KMS key 사용
AWS Backup: Vault Lock 옵션
```

### Vault Lock
```bash
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name bkurs-vault-aps2 \
  --min-retention-days 7 \
  --max-retention-days 365 \
  --changeable-for-days 3
```

→ 7년 compliance retention 등.

## 12. CloudWatch metric

```
SuccessfulRequestLatency (CreateBackup)
SystemErrors
ContinuousBackupsCount
```

## 13. Restore Best Practice

```
1. Test restore monthly
2. Backup 의 정기 monitor
3. Cross-region copy 활성
4. AWS Backup 의 Vault Lock 활용
5. Restore 후 application config 변경 (table name)
```

## 14. PITR vs Streams 의 결합

```
PITR:    point-in-time 복원 (실수 delete recovery)
Streams: real-time event processing
```

→ 둘은 다른 목적. 같이 사용 권장.

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| PITR 비활성 → 실수 복구 불가 | 항상 활성 |
| Restore = overwrite 가정 | 새 table |
| GSI / Streams / Auto Scaling 복원 안 됨 | 수동 재설정 |
| Cross-region 없음 → DR ↓ | AWS Backup CopyActions |
| Backup 의 retention 무한 | lifecycle (cold + delete) |
| Vault Lock 없음 → 실수 영구 손실 | enable |

## 16. 권장 — bkurs-be DynamoDB

```
Critical table (orders, payments):
- PITR enabled (35d)
- AWS Backup daily (cron 18:00)
- Cross-region copy (ap-northeast-1)
- Vault Lock 7y compliance
- Streams enabled (audit)

General table (sessions, cache):
- PITR enabled
- AWS Backup weekly (or none)
```

## 17. 다음

- ▶ [[../practice/01-table-with-gsi]]
- ▶ [[../practice/02-streams-lambda-pipeline]]

## 18. 영상 / 자료

| 자료 |
| --- |
| [PITR](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery.html) |
| [On-Demand Backup](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/BackupRestore.html) |
| [S3 Export](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport.HowItWorks.html) |

## 19. 참고

- [[../dynamodb]]
- [[01-streams-and-cdc]]
- [[../../rds/02-operations/01-backup-and-pitr]]
