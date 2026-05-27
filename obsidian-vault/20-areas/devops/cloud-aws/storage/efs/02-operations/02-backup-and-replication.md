---
title: "EFS 02-2 — AWS Backup + Cross-region Replication"
kind: knowledge
status: current
created_at: 2026-05-22T19:42:00+09:00
tags: [aws, efs, operations, backup, aws-backup, replication, dr]
home_hub: efs
---

# EFS Backup + Replication

> AWS Backup (automatic / managed) + Cross-region Replication (1-click DR). Native incremental.

## 1. AWS Backup vs Native

| | AWS Backup | EFS Native (legacy) |
| --- | --- | --- |
| Schedule | declarative | manual |
| Multi-service (EFS + EBS + RDS + ...) | ✓ | ✗ |
| Vault + Lock | ✓ | ✗ |
| Cross-region copy | ✓ | manual |
| Cost | per backup GB·월 | n/a |
| 권장 | ✓ | EOL |

## 2. AWS Backup 설정

```bash
# Backup Vault
aws backup create-backup-vault --backup-vault-name bkurs-efs-vault \
  --encryption-key-arn arn:aws:kms:ap-northeast-2:ACCT:key/abc

# Backup Plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "bkurs-efs-daily",
  "Rules": [{
    "RuleName": "daily",
    "TargetBackupVaultName": "bkurs-efs-vault",
    "ScheduleExpression": "cron(0 18 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 240,
    "Lifecycle": {
      "MoveToColdStorageAfterDays": 30,
      "DeleteAfterDays": 365
    },
    "CopyActions": [{
      "DestinationBackupVaultArn": "arn:aws:backup:ap-northeast-1:ACCT:backup-vault:bkurs-efs-vault-dr",
      "Lifecycle": { "DeleteAfterDays": 365 }
    }]
  }]
}'

# Resource Assignment
aws backup create-backup-selection --backup-plan-id $PLAN_ID \
  --backup-selection '{
    "SelectionName": "efs-tagged",
    "IamRoleArn": "arn:aws:iam::ACCT:role/AWSBackupDefaultServiceRole",
    "Resources": ["arn:aws:elasticfilesystem:ap-northeast-2:ACCT:file-system/*"],
    "Conditions": {
      "StringEquals": [{ "ConditionKey": "aws:ResourceTag/Backup", "ConditionValue": "daily" }]
    }
  }'
```

## 3. Backup tag 적용

```bash
aws efs tag-resource --resource-id fs-xxx \
  --tags Key=Backup,Value=daily
```

→ Plan 의 selection filter 와 매칭 시 자동 backup.

## 4. Vault Lock (immutable backup)

```bash
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name bkurs-efs-vault \
  --min-retention-days 30 \
  --max-retention-days 365 \
  --changeable-for-days 3
```

→ 3일 안 변경 가능, 그 후 lock. 7년 compliance 등 활용.

## 5. 복원

```bash
# Backup list
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name bkurs-efs-vault \
  --query 'RecoveryPoints[].{Arn:RecoveryPointArn,Time:CreationDate,Size:BackupSizeInBytes}'

# Restore (new FS 또는 existing FS path)
aws backup start-restore-job \
  --recovery-point-arn arn:aws:backup:ap-northeast-2:ACCT:recovery-point:xxx \
  --resource-type EFS \
  --iam-role-arn arn:iam::ACCT:role/AWSBackupDefaultServiceRole \
  --metadata '{
    "file-system-id": "fs-xxx",
    "Encrypted": "true",
    "KmsKeyId": "alias/efs",
    "PerformanceMode": "generalPurpose",
    "newFileSystem": "true",
    "CreationToken": "restored-$(date +%s)",
    "ItemsToRestore": "[{\"SourcePath\":\"/\",\"DestinationPath\":\"/\"}]"
  }'
```

## 6. EFS Cross-region Replication

```bash
# Source 의 setup
aws efs create-replication-configuration --source-file-system-id fs-src \
  --destinations Region=ap-northeast-1,KmsKeyId=arn:aws:kms:ap-northeast-1:ACCT:key/abc

# 또는 같은 region 의 다른 AZ (One Zone)
aws efs create-replication-configuration --source-file-system-id fs-src \
  --destinations AvailabilityZoneName=ap-northeast-2c
```

특징:
- async, 보통 분 단위
- RPO (Recovery Point Objective) ~15분
- Destination 은 read-only
- failover 시 promote 가능

## 7. Replication 의 RPO / RTO

| | |
| --- | --- |
| RPO | ~15분 (last sync) |
| RTO | promote ~수분 (DNS 변경) |

## 8. Failover

```bash
# Destination 을 standalone 으로 promote
aws efs delete-replication-configuration --source-file-system-id fs-src

# Destination FS 는 그대로 사용 가능 (다른 region 의 source 로 새 replication 설정 가능)
```

## 9. Backup vs Replication 비교

| | Backup | Replication |
| --- | --- | --- |
| 목적 | point-in-time recovery | DR / active failover |
| RPO | hours | 15분 |
| RTO | hours (restore 시간) | 분 |
| 비용 | $0.05/GB·월 + cold storage 추가 | dest storage + DT |
| Immutability | Vault Lock | ✗ |
| Cross-region | ✓ | ✓ |
| Use | compliance, accidental delete | active DR |

→ **둘 다 활용 권장**.

## 10. EFS-to-EFS replication

```bash
# Source → 다른 EFS
aws efs create-replication-configuration --source-file-system-id fs-src \
  --destinations Region=ap-northeast-1
```

> One-to-one. fan-out 안 됨.

## 11. AWS Backup 비용

| | |
| --- | --- |
| Backup storage (warm) | $0.05/GB·월 |
| Cold storage | $0.01/GB·월 (90일 이상) |
| Cross-region copy | DT $0.09/GB |
| Restore | 무료 |

## 12. EventBridge 통합

```yaml
EventPattern:
  source: [aws.backup]
  detail-type: ["Backup Job State Change", "Restore Job State Change"]
  detail:
    state: [COMPLETED, FAILED]
Targets:
  - Arn: arn:sns:...:bkurs-alerts
```

## 13. Replication CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name TimeSinceLastSync \
  --dimensions Name=FileSystemId,Value=fs-src \
  --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time   $(date -u                 +"%Y-%m-%dT%H:%M:%SZ") \
  --period 60 --statistics Maximum
```

| Metric | 의미 |
| --- | --- |
| `TimeSinceLastSync` | 마지막 sync 이후 시간 |
| `ReplicationLag` | source ↔ destination 차이 |

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Backup 없음 → 실수 삭제 = 영구 손실 | 항상 enable |
| Replication 만 → ransomware 도 copy | Backup + Vault Lock |
| Vault Lock changeable 0 → 실수 못 풀음 | changeable-for-days 3+ |
| Cross-region cost 무시 | DR 만 |
| Backup IAM role 권한 부족 | AWSBackupDefaultServiceRole |
| TimeSinceLastSync 큰 값 → replication lag | alarm + alert |

## 15. 권장 — bkurs-be EFS

```
Standard FS:
- AWS Backup daily (cron 18:00 KST)
- Cold storage after 30d
- Delete after 365d
- Cross-region copy to ap-northeast-1
- Vault Lock 7d changeable + 365 day min

Critical FS:
- Replication to ap-northeast-1 (15분 RPO)
- + Backup (immutable Vault Lock)
- + Object Lock 별도 S3 archive
```

## 16. 다음

- ▶ [[03-mount-and-csi]]
- ▶ [[../practice/02-lifecycle-and-replication]]
- ▶ [[../../security/kms/kms]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [AWS Backup](https://docs.aws.amazon.com/aws-backup/) |
| [EFS Replication](https://docs.aws.amazon.com/efs/latest/ug/efs-replication.html) |

## 18. 참고

- [[../efs]]
- [[01-access-points-and-iam]]
- [[../../security/kms/kms]]
