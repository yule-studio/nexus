---
title: "EFS 실습 2 — Lifecycle (Standard → IA → Archive) + Cross-region Replication"
kind: practice
status: current
created_at: 2026-05-22T19:48:00+09:00
tags: [aws, efs, practice, lifecycle, replication, dr, backup]
home_hub: efs
---

# 실습 2 — Lifecycle + Cross-region Replication

> 목표: Lifecycle (Standard → IA → Archive) + AWS Backup + Cross-region Replication + Failover 시뮬. 45분.

## 1. Source EFS

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

KEY_ID=$(aws kms create-key --description "lab efs lifecycle" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-efs-lifecycle --target-key-id $KEY_ID

# Source FS (Seoul)
SRC=$(aws efs create-file-system \
  --creation-token lab-src-$(date +%s) \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted --kms-key-id alias/lab-efs-lifecycle \
  --tags Key=Name,Value=lab-src Key=Environment,Value=prod \
  --query 'FileSystemId' --output text)
```

## 2. Lifecycle Policy

```bash
aws efs put-lifecycle-configuration --file-system-id $SRC \
  --lifecycle-policies '[
    { "TransitionToIA": "AFTER_30_DAYS" },
    { "TransitionToArchive": "AFTER_90_DAYS" },
    { "TransitionToPrimaryStorageClass": "AFTER_1_ACCESS" }
  ]'

aws efs describe-lifecycle-configuration --file-system-id $SRC
```

## 3. AWS Backup Plan

```bash
# Backup Vault (Seoul, KMS)
aws backup create-backup-vault --backup-vault-name lab-vault-aps2 \
  --encryption-key-arn arn:aws:kms:ap-northeast-2:$ACCT:key/$KEY_ID

# DR Vault (Tokyo)
KEY_DR=$(aws kms create-key --description "lab efs DR" --region ap-northeast-1 \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-efs-dr --target-key-id $KEY_DR --region ap-northeast-1
aws backup create-backup-vault --backup-vault-name lab-vault-apn1 \
  --encryption-key-arn arn:aws:kms:ap-northeast-1:$ACCT:key/$KEY_DR \
  --region ap-northeast-1

# Backup Plan
aws backup create-backup-plan --backup-plan "{
  \"BackupPlanName\": \"lab-efs-plan\",
  \"Rules\": [{
    \"RuleName\": \"daily\",
    \"TargetBackupVaultName\": \"lab-vault-aps2\",
    \"ScheduleExpression\": \"cron(0 18 * * ? *)\",
    \"StartWindowMinutes\": 60,
    \"CompletionWindowMinutes\": 240,
    \"Lifecycle\": {
      \"MoveToColdStorageAfterDays\": 30,
      \"DeleteAfterDays\": 365
    },
    \"CopyActions\": [{
      \"DestinationBackupVaultArn\": \"arn:aws:backup:ap-northeast-1:$ACCT:backup-vault:lab-vault-apn1\",
      \"Lifecycle\": { \"DeleteAfterDays\": 365 }
    }]
  }]
}"

PLAN_ID=$(aws backup list-backup-plans --query 'BackupPlansList[?BackupPlanName==`lab-efs-plan`].BackupPlanId' --output text)

# Selection (tag-based)
aws backup create-backup-selection --backup-plan-id $PLAN_ID \
  --backup-selection "{
    \"SelectionName\": \"efs-prod\",
    \"IamRoleArn\": \"arn:aws:iam::$ACCT:role/service-role/AWSBackupDefaultServiceRole\",
    \"ListOfTags\": [{
      \"ConditionType\": \"STRINGEQUALS\",
      \"ConditionKey\": \"Environment\",
      \"ConditionValue\": \"prod\"
    }]
  }"
```

> `AWSBackupDefaultServiceRole` — Console 첫 사용 시 자동 생성됨. CLI 첫 사용 시 수동 생성:
```bash
aws iam create-role --role-name service-role/AWSBackupDefaultServiceRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"backup.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name service-role/AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup
aws iam attach-role-policy --role-name service-role/AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores
```

## 4. Vault Lock (compliance)

```bash
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name lab-vault-aps2 \
  --min-retention-days 7 \
  --max-retention-days 365 \
  --changeable-for-days 3
```

→ 3일 안 변경 가능, 그 후 lock.

## 5. Cross-region Replication (EFS native)

```bash
aws efs create-replication-configuration --source-file-system-id $SRC \
  --destinations Region=ap-northeast-1,KmsKeyId=arn:aws:kms:ap-northeast-1:$ACCT:key/$KEY_DR

# Destination FS ID 확인
DST=$(aws efs describe-replication-configurations --file-system-id $SRC \
  --query 'Replications[0].Destinations[0].FileSystemId' --output text)
echo "DST=$DST"
```

## 6. Test data 생성 + replication 검증

```bash
# 실습 1 의 EC2 인스턴스 재사용 또는 새로
# Mount Target 같은 SG/VPC

# Write
aws ssm send-command --instance-ids $INST \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"mkdir -p /mnt/src\",
    \"mount -t efs -o tls $SRC:/ /mnt/src\",
    \"for i in {1..100}; do echo data-\$i > /mnt/src/file-\$i.txt; done\",
    \"ls /mnt/src/ | wc -l\"
  ]"

# Replication metric
sleep 60
aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name TimeSinceLastSync \
  --dimensions Name=FileSystemId,Value=$SRC \
  --period 60 --statistics Maximum --start-time $(date -u -v -10M +"%Y-%m-%dT%H:%M:%SZ") --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

## 7. Backup 즉시 실행 (수동)

```bash
aws backup start-backup-job \
  --backup-vault-name lab-vault-aps2 \
  --resource-arn arn:aws:elasticfilesystem:ap-northeast-2:$ACCT:file-system/$SRC \
  --iam-role-arn arn:aws:iam::$ACCT:role/service-role/AWSBackupDefaultServiceRole

# Job status
aws backup list-backup-jobs --query 'BackupJobs[?ResourceArn==`arn:aws:elasticfilesystem:ap-northeast-2:'$ACCT':file-system/'$SRC'`]'
```

## 8. Failover 시뮬

```bash
# Destination 을 standalone 으로 promote
aws efs delete-replication-configuration --source-file-system-id $SRC

# Destination 은 그대로 사용 가능 (다른 region 에서)
aws efs describe-file-systems --file-system-id $DST --region ap-northeast-1

# DR region 에서 Mount Target 추가 (이전에 없었다면)
VPC_DR=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --region ap-northeast-1 --query 'Vpcs[0].VpcId' --output text)
SUBNET_DR=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_DR --region ap-northeast-1 --query 'Subnets[0].SubnetId' --output text)

aws efs create-mount-target --file-system-id $DST \
  --subnet-id $SUBNET_DR --security-groups sg-xxx --region ap-northeast-1
```

## 9. Restore 시뮬

```bash
# Backup list
RECOVERY=$(aws backup list-recovery-points-by-backup-vault --backup-vault-name lab-vault-aps2 \
  --query 'RecoveryPoints[0].RecoveryPointArn' --output text)

# Restore (새 FS)
aws backup start-restore-job \
  --recovery-point-arn $RECOVERY \
  --iam-role-arn arn:aws:iam::$ACCT:role/service-role/AWSBackupDefaultServiceRole \
  --resource-type EFS \
  --metadata "{
    \"file-system-id\": \"$SRC\",
    \"Encrypted\": \"true\",
    \"KmsKeyId\": \"arn:aws:kms:ap-northeast-2:$ACCT:key/$KEY_ID\",
    \"PerformanceMode\": \"generalPurpose\",
    \"CreationToken\": \"restored-$(date +%s)\",
    \"newFileSystem\": \"true\",
    \"ItemsToRestore\": \"[{\\\"SourcePath\\\":\\\"/\\\",\\\"DestinationPath\\\":\\\"/\\\"}]\"
  }"
```

## 10. Lifecycle 결과 시뮬

EFS lifecycle 은 30일 단위라 즉시 확인 어려움. CloudWatch 의 `StorageBytes` 의 StorageClass dimension 확인:
```bash
aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name StorageBytes \
  --dimensions Name=FileSystemId,Value=$SRC Name=StorageClass,Value=Standard \
  --period 86400 --statistics Average \
  --start-time $(date -u -v -7d +"%Y-%m-%dT%H:%M:%SZ") --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")

aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name StorageBytes \
  --dimensions Name=FileSystemId,Value=$SRC Name=StorageClass,Value=IA \
  --period 86400 --statistics Average \
  --start-time $(date -u -v -7d +"%Y-%m-%dT%H:%M:%SZ") --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

## 11. EventBridge — Backup 알림

```bash
SNS_ARN=$(aws sns create-topic --name lab-backup-alerts --query 'TopicArn' --output text)

aws events put-rule --name lab-backup-state \
  --event-pattern '{
    "source": ["aws.backup"],
    "detail-type": ["Backup Job State Change","Restore Job State Change"],
    "detail": { "state": ["COMPLETED","FAILED"] }
  }'

aws events put-targets --rule lab-backup-state \
  --targets "Id=1,Arn=$SNS_ARN"
```

## 12. Clean up

```bash
# Replication
aws efs delete-replication-configuration --source-file-system-id $SRC 2>/dev/null

# Mount Targets (양쪽)
for MT in $(aws efs describe-mount-targets --file-system-id $SRC --query 'MountTargets[].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id $MT
done

for MT in $(aws efs describe-mount-targets --file-system-id $DST --region ap-northeast-1 --query 'MountTargets[].MountTargetId' --output text 2>/dev/null); do
  aws efs delete-mount-target --mount-target-id $MT --region ap-northeast-1
done

sleep 30

aws efs delete-file-system --file-system-id $SRC
aws efs delete-file-system --file-system-id $DST --region ap-northeast-1 2>/dev/null

# Backup
aws backup delete-backup-selection --backup-plan-id $PLAN_ID \
  --selection-id $(aws backup list-backup-selections --backup-plan-id $PLAN_ID --query 'BackupSelectionsList[0].SelectionId' --output text)
aws backup delete-backup-plan --backup-plan-id $PLAN_ID

aws backup delete-backup-vault --backup-vault-name lab-vault-aps2 2>/dev/null
aws backup delete-backup-vault --backup-vault-name lab-vault-apn1 --region ap-northeast-1 2>/dev/null

aws events remove-targets --rule lab-backup-state --ids 1
aws events delete-rule --name lab-backup-state
aws sns delete-topic --topic-arn $SNS_ARN

aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/lab-efs-lifecycle
aws kms schedule-key-deletion --key-id $KEY_DR --pending-window-in-days 7 --region ap-northeast-1
aws kms delete-alias --alias-name alias/lab-efs-dr --region ap-northeast-1
```

## 13. 학습 포인트

- Lifecycle 의 3 stage (Standard → IA → Archive)
- TransitionToPrimaryStorageClass=AFTER_1_ACCESS
- AWS Backup + Vault Lock (immutable)
- Cross-region Backup Copy + Replication (둘 다 활용)
- EFS Replication 의 RPO ~15분
- Failover (delete-replication-configuration)
- Restore (new FS)
- EventBridge → SNS 알림

## 14. 다음

- ▶ [[../efs]]
- ▶ [[../../../database/rds/rds]]

## 15. 참고

- [[../efs]]
- [[../01-fundamentals/02-storage-classes-and-lifecycle]]
- [[../02-operations/02-backup-and-replication]]
