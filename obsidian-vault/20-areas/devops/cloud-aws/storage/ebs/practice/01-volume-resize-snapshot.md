---
title: "EBS 실습 1 — Volume 동작 중 Resize + Snapshot + DLM + AMI"
kind: practice
status: current
created_at: 2026-05-22T19:27:00+09:00
tags: [aws, ebs, practice, resize, snapshot, dlm, ami]
home_hub: ebs
---

# 실습 1 — Volume Resize + Snapshot + DLM

> 목표: gp3 100GB 생성 → attach → 200GB 확장 → snapshot → DLM 자동 → AMI. 30분.

## 1. EC2 + EBS Volume

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNET=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[0].SubnetId' --output text)
AZ=$(aws ec2 describe-subnets --subnet-ids $SUBNET --query 'Subnets[0].AvailabilityZone' --output text)

# SG
SG=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-ebs \
  --description "ebs lab" --query GroupId --output text)

# SSM Instance Profile (실습 1 의 FirstEc2Profile 재사용)
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

# EC2 인스턴스
INST=$(aws ec2 run-instances --image-id $AMI --instance-type t4g.small \
  --subnet-id $SUBNET --security-group-ids $SG \
  --iam-instance-profile Name=FirstEc2Profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-ebs}]' \
  --query 'Instances[0].InstanceId' --output text)

sleep 60

# 추가 Data Volume (100 GB gp3)
VOL=$(aws ec2 create-volume \
  --availability-zone $AZ \
  --size 100 --volume-type gp3 \
  --iops 3000 --throughput 125 \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=lab-data},{Key=Backup,Value=daily}]' \
  --query 'VolumeId' --output text)

# Attach
aws ec2 wait volume-available --volume-ids $VOL
aws ec2 attach-volume --volume-id $VOL --instance-id $INST --device /dev/sdf
sleep 30
```

## 2. Mount

```bash
aws ssm start-session --target $INST

# Inside instance
lsblk
# nvme0n1  30G  (root)
# nvme1n1 100G  (data)

sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data
df -h /data
# /dev/nvme1n1   100G

# fstab (NVMe = UUID 권장)
UUID=$(sudo blkid -s UUID -o value /dev/nvme1n1)
echo "UUID=$UUID /data xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Test write
echo "test" | sudo tee /data/test.txt
exit
```

## 3. Performance baseline

```bash
aws ssm start-session --target $INST
sudo dnf install -y fio
sudo fio --filename=/data/fio-test --rw=randread --bs=4k --size=1G \
  --numjobs=4 --iodepth=64 --runtime=30 --time_based \
  --name=baseline --ioengine=libaio --direct=1 --group_reporting

# IOPS=3000 (free baseline)
exit
```

## 4. Resize (zero-downtime)

```bash
# Volume modify
aws ec2 modify-volume --volume-id $VOL --size 200 --iops 8000 --throughput 250

# 진행 확인
watch -n 5 "aws ec2 describe-volumes-modifications --volume-ids $VOL --query 'VolumesModifications[0].{State:ModificationState,Progress:Progress}'"

# 완료 후 (~수분)
aws ssm start-session --target $INST

# Inside
sudo growpart /dev/nvme1n1 1 2>/dev/null || true   # XFS no partition, skip
sudo xfs_growfs /data
df -h /data
# /dev/nvme1n1   200G    (확장됨)

# Re-benchmark
sudo fio --filename=/data/fio-test --rw=randread --bs=4k --size=1G \
  --numjobs=4 --iodepth=64 --runtime=30 --time_based \
  --name=resized --ioengine=libaio --direct=1 --group_reporting

# IOPS ~8000 (provisioned)
exit
```

## 5. Snapshot 생성

```bash
SNAP=$(aws ec2 create-snapshot --volume-id $VOL \
  --description "lab-baseline" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=lab-data-snap}]' \
  --query 'SnapshotId' --output text)

aws ec2 wait snapshot-completed --snapshot-ids $SNAP
```

## 6. DLM Lifecycle Policy

```bash
# DLM role (재사용 가능)
aws iam create-role --role-name AWSDataLifecycleManagerDefaultRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"dlm.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true

aws iam attach-role-policy --role-name AWSDataLifecycleManagerDefaultRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSDataLifecycleManagerServiceRole \
  2>/dev/null || true

ACCT=$(aws sts get-caller-identity --query Account --output text)

aws dlm create-lifecycle-policy \
  --execution-role-arn arn:aws:iam::$ACCT:role/AWSDataLifecycleManagerDefaultRole \
  --description "lab daily snapshot" \
  --state ENABLED \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "TargetTags": [{ "Key": "Backup", "Value": "daily" }],
    "Schedules": [{
      "Name": "daily",
      "CreateRule": { "Interval": 24, "IntervalUnit": "HOURS" },
      "RetainRule": { "Count": 7 },
      "TagsToAdd": [{ "Key": "DLM", "Value": "true" }]
    }]
  }'
```

→ 매일 자동 snapshot, 7일 보존, `Backup=daily` tag 있는 volume.

## 7. AMI 생성 (running instance)

```bash
aws ec2 create-image --instance-id $INST \
  --name "lab-app-v1.0.0" \
  --description "lab AMI v1" \
  --no-reboot

aws ec2 wait image-available --image-ids ami-...   # 생성 대기

# Confirm
aws ec2 describe-images --owners self \
  --filters Name=name,Values=lab-app-v1.0.0 \
  --query 'Images[0].{ImageId:ImageId,Snapshots:BlockDeviceMappings[].Ebs.SnapshotId}'
```

> 데이터 volume 도 snapshot 됨. AMI launch 시 같은 구조로 부팅.

## 8. Snapshot 복원 (다른 AZ)

```bash
AZ_OTHER=$(aws ec2 describe-availability-zones \
  --query "AvailabilityZones[?ZoneName!='$AZ']|[0].ZoneName" --output text)

VOL_NEW=$(aws ec2 create-volume --snapshot-id $SNAP \
  --availability-zone $AZ_OTHER --volume-type gp3 \
  --query 'VolumeId' --output text)

# Different AZ 의 새 instance 에 attach (생략)
```

## 9. Recycle Bin

```bash
aws rbin create-rule --resource-type EBS_SNAPSHOT \
  --retention-period RetentionPeriodValue=7,RetentionPeriodUnit=DAYS \
  --description "lab 7d" \
  --resource-tags ResourceTagKey=Name,ResourceTagValue=lab-data-snap
```

→ 실수 삭제 7일 보호.

## 10. Snapshot Archive Tier

```bash
# 90일 이상 보존 시 archive (75% 절감)
aws ec2 modify-snapshot-tier --snapshot-id $SNAP --storage-tier archive

aws ec2 describe-snapshot-tier-status --filters Name=snapshot-id,Values=$SNAP \
  --query 'SnapshotTierStatuses[0]'
```

> 복원 시 24-72h. 자주 사용 X.

## 11. CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=$VOL \
  --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time   $(date -u                 +"%Y-%m-%dT%H:%M:%SZ") \
  --period 60 --statistics Sum,Average \
  --query 'Datapoints | sort_by(@, &Timestamp)'
```

## 12. Clean up

```bash
# DLM
DLM_ID=$(aws dlm get-lifecycle-policies --query 'Policies[?Description==`lab daily snapshot`].PolicyId' --output text)
aws dlm delete-lifecycle-policy --policy-id $DLM_ID

# Snapshot
aws ec2 modify-snapshot-tier --snapshot-id $SNAP --storage-tier standard 2>/dev/null || true
aws ec2 delete-snapshot --snapshot-id $SNAP

# AMI
AMI_LAB=$(aws ec2 describe-images --owners self \
  --filters Name=name,Values=lab-app-v1.0.0 \
  --query 'Images[0].ImageId' --output text)
SNAPS=$(aws ec2 describe-images --image-ids $AMI_LAB \
  --query 'Images[0].BlockDeviceMappings[].Ebs.SnapshotId' --output text)
aws ec2 deregister-image --image-id $AMI_LAB
for s in $SNAPS; do
  aws ec2 delete-snapshot --snapshot-id $s 2>/dev/null
done

# Volume
aws ec2 detach-volume --volume-id $VOL --force 2>/dev/null
sleep 10
aws ec2 delete-volume --volume-id $VOL

aws ec2 delete-volume --volume-id $VOL_NEW 2>/dev/null

# Instance
aws ec2 terminate-instances --instance-ids $INST
aws ec2 wait instance-terminated --instance-ids $INST
aws ec2 delete-security-group --group-id $SG

# Recycle Bin (옵션 — 실습 후 disable)
RBIN=$(aws rbin list-rules --resource-type EBS_SNAPSHOT --query 'Rules[?Description==`lab 7d`].Identifier' --output text)
aws rbin delete-rule --identifier $RBIN 2>/dev/null
```

## 13. 학습 포인트

- `aws ec2 modify-volume` zero-downtime resize (6h 1회)
- `growpart` + `xfs_growfs` filesystem 확장
- gp3 의 free baseline + provisioned IOPS/Throughput
- DLM tag-based 자동 snapshot
- AMI = boot + data snapshot
- Cross-AZ recovery (snapshot)
- Archive Tier (75% 비용 절감)
- Recycle Bin (실수 보호)

## 14. 다음

- ▶ [[02-encrypted-multi-attach]]
- ▶ [[../01-fundamentals/02-snapshot-and-ami]]
- ▶ [[../../../compute/ec2/03-storage/03-snapshots-ami-lifecycle]]

## 15. 참고

- [[../ebs]]
- [[../02-operations/03-performance-and-monitoring]]
