---
title: "EBS 실습 2 — Encrypted io2 Multi-Attach + GFS2 Cluster"
kind: practice
status: current
created_at: 2026-05-22T19:29:00+09:00
tags: [aws, ebs, practice, multi-attach, gfs2, encryption, cluster]
home_hub: ebs
---

# 실습 2 — io2 Multi-Attach + GFS2

> 목표: io2 encrypted Multi-Attach + 2 EC2 + GFS2 cluster filesystem. 60분.

> 실습 — 실제 RHEL 9 cluster 설정 필요. AL2023 / Ubuntu 는 GFS2 패키지 부재.

## 1. KMS Key (CMK)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

KEY_ID=$(aws kms create-key --description "lab ebs encryption" \
  --key-spec SYMMETRIC_DEFAULT --key-usage ENCRYPT_DECRYPT \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-ebs --target-key-id $KEY_ID

# Default encryption account-wide
aws ec2 enable-ebs-encryption-by-default
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/lab-ebs
```

## 2. io2 Multi-Attach Volume

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNET=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[0].SubnetId' --output text)
AZ=$(aws ec2 describe-subnets --subnet-ids $SUBNET --query 'Subnets[0].AvailabilityZone' --output text)

# io2 Multi-Attach 활성
VOL=$(aws ec2 create-volume \
  --availability-zone $AZ \
  --size 100 --volume-type io2 \
  --iops 10000 \
  --multi-attach-enabled \
  --encrypted --kms-key-id alias/lab-ebs \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=lab-shared}]' \
  --query 'VolumeId' --output text)

aws ec2 wait volume-available --volume-ids $VOL
```

## 3. RHEL 9 Marketplace AMI (subscribe 필요)

```bash
# RHEL 9 ARM
AMI=$(aws ec2 describe-images --owners 309956199498 \
  --filters Name=name,Values="RHEL-9*-arm64-*" Name=state,Values=available \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text)

# 또는 직접 marketplace AMI 선택
```

## 4. SG (Multi-Attach 같은 SG 공유)

```bash
SG=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-cluster \
  --description "cluster" --query GroupId --output text)

# Inter-node Pacemaker / Corosync
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 2224 --source-group $SG
aws ec2 authorize-security-group-ingress --group-id $SG --protocol udp --port 5404-5405 --source-group $SG
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 21064 --source-group $SG
# SSM access
```

## 5. EC2 인스턴스 2개

```bash
for i in 1 2; do
  aws ec2 run-instances --image-id $AMI --instance-type m6g.large \
    --subnet-id $SUBNET --security-group-ids $SG \
    --iam-instance-profile Name=FirstEc2Profile \
    --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=lab-node-$i}]" \
    --user-data "$(cat <<'EOF'
#!/bin/bash
hostnamectl set-hostname node-$INSTANCE_ID
EOF
)"
done

sleep 90

INST1=$(aws ec2 describe-instances --filters Name=tag:Name,Values=lab-node-1 Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
INST2=$(aws ec2 describe-instances --filters Name=tag:Name,Values=lab-node-2 Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
```

## 6. Volume attach to both

```bash
aws ec2 attach-volume --volume-id $VOL --instance-id $INST1 --device /dev/sdf
sleep 10
aws ec2 attach-volume --volume-id $VOL --instance-id $INST2 --device /dev/sdf
sleep 10

# Confirm
aws ec2 describe-volumes --volume-ids $VOL \
  --query 'Volumes[0].{State:State,Attachments:Attachments[].InstanceId}'
# 두 instance 모두 attached
```

## 7. Pacemaker cluster (node1 에서)

```bash
aws ssm start-session --target $INST1

# Install
sudo dnf install -y pcs pacemaker corosync resource-agents fence-agents-aws gfs2-utils lvm2-cluster dlm

# Auth (cluster)
sudo passwd hacluster <<EOF
clusterpass
clusterpass
EOF

sudo systemctl enable --now pcsd

# /etc/hosts (양 노드)
sudo tee -a /etc/hosts <<EOF
10.0.x.x node-1
10.0.y.y node-2
EOF

# Auth between nodes
sudo pcs host auth -u hacluster -p clusterpass node-1 node-2

# Cluster setup
sudo pcs cluster setup --name bkurs-cluster node-1 node-2
sudo pcs cluster start --all
sudo pcs cluster enable --all
```

(node2 도 같은 작업 — pcs host auth 후 join)

## 8. DLM + clvmd

```bash
sudo pcs resource create dlm ocf:pacemaker:controld \
  op monitor interval=30s on-fail=fence clone meta interleave=true ordered=true

sudo pcs resource create clvmd ocf:heartbeat:clvm \
  op monitor interval=30s on-fail=fence clone meta interleave=true ordered=true

sudo pcs constraint order start dlm-clone then clvmd-clone
sudo pcs constraint colocation add clvmd-clone with dlm-clone
```

## 9. GFS2 Format

```bash
# Volume 의 NVMe device 확인
sudo nvme list
# /dev/nvme1n1 = io2 Multi-Attach

# LVM (cluster shared)
sudo pvcreate /dev/nvme1n1
sudo vgcreate -Ay -cy data-vg /dev/nvme1n1
sudo lvcreate -n data-lv -l 100%FREE data-vg

# GFS2 format (-j journals = node 수 + 1)
sudo mkfs.gfs2 -j 3 -p lock_dlm -t bkurs-cluster:data /dev/data-vg/data-lv
```

## 10. Mount (양 노드)

```bash
sudo mkdir /shared
sudo pcs resource create gfs2-share Filesystem \
  device=/dev/data-vg/data-lv \
  directory=/shared fstype=gfs2 \
  op monitor interval=30s on-fail=fence \
  clone interleave=true

# Both nodes
df -h /shared
# /dev/mapper/data--vg-data--lv   100G   /shared

# Write from node 1
echo "from node 1" | sudo tee /shared/test.txt

# Read from node 2
aws ssm start-session --target $INST2
cat /shared/test.txt
# from node 1
```

## 11. Concurrent write test

```bash
# Node 1
for i in $(seq 1 100); do
  echo "node-1: $i" | sudo tee -a /shared/log.txt
done

# 동시 Node 2
for i in $(seq 1 100); do
  echo "node-2: $i" | sudo tee -a /shared/log.txt
done

# Confirm (GFS2 의 locking 으로 corruption 없음)
wc -l /shared/log.txt
# 200 lines (모두 보존)
```

## 12. Fencing (필수, production)

split-brain 방지:
```bash
sudo pcs stonith create aws-fence fence_aws \
  region=ap-northeast-2 access_key=$AWS_ACCESS_KEY \
  secret_key=$AWS_SECRET_KEY \
  pcmk_host_map="node-1:i-xxx;node-2:i-yyy"
```

→ 한 노드가 fail / partition 되면 다른 노드가 EC2 강제 종료.

## 13. Failover 시뮬

```bash
# Node 1 의 cluster service 중단
aws ssm start-session --target $INST1
sudo pcs cluster stop --force

# Node 2 에서 shared volume 여전히 접근 가능
aws ssm start-session --target $INST2
ls /shared
# test.txt log.txt (정상)

# Node 1 의 GFS2 lock 해제 + Node 2 가 단독 작동
```

## 14. Snapshot (Multi-Attach 도 가능)

```bash
SNAP=$(aws ec2 create-snapshot --volume-id $VOL \
  --description "multi-attach crash-consistent" \
  --query 'SnapshotId' --output text)

aws ec2 wait snapshot-completed --snapshot-ids $SNAP

# 새 volume 복원
aws ec2 create-volume --snapshot-id $SNAP \
  --availability-zone $AZ --volume-type io2 \
  --iops 10000 --multi-attach-enabled --encrypted
```

## 15. Clean up

```bash
# Node 들 의 cluster stop
for I in $INST1 $INST2; do
  aws ssm send-command --instance-ids $I \
    --document-name AWS-RunShellScript \
    --parameters 'commands=["sudo pcs cluster stop --force","sudo umount /shared","sudo pcs cluster destroy"]'
done

# EC2 종료
aws ec2 terminate-instances --instance-ids $INST1 $INST2
aws ec2 wait instance-terminated --instance-ids $INST1 $INST2

# Volume detach + delete
aws ec2 delete-volume --volume-id $VOL

# Snapshot
aws ec2 delete-snapshot --snapshot-id $SNAP

# SG
aws ec2 delete-security-group --group-id $SG

# KMS key
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/lab-ebs
```

## 16. 학습 포인트

- io2 Multi-Attach 활성 (Nitro + 같은 AZ + 16 max)
- KMS CMK + Default Encryption
- GFS2 cluster filesystem 의 distributed lock manager (DLM)
- Pacemaker + Corosync (cluster resource manager)
- Fencing (STONITH) = split-brain 방지
- LVM cluster shared
- Crash-consistent snapshot (모든 node 의 동시 상태)

## 17. 권장 — bkurs-be 와의 관계

```
Multi-Attach 는 매우 특수 (Oracle RAC, SAP HANA 등).
일반 web/API workload = ECS Fargate + EFS / S3.
DB = RDS / Aurora (managed) — EBS 직접 다루지 않음.
```

→ 이 실습은 학습 / 특수 use case 용.

## 18. 다음

- ▶ [[../ebs]]
- ▶ [[../../efs/efs]]
- ▶ [[../02-operations/02-multi-attach-and-fast-snapshot-restore]]

## 19. 참고

- [[../ebs]]
- [[01-volume-resize-snapshot]]
- [[../02-operations/01-encryption-kms]]
- [[../02-operations/02-multi-attach-and-fast-snapshot-restore]]
