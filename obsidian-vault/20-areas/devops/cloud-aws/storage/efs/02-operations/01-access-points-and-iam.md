---
title: "EFS 02-1 — Access Points + IAM Authorization"
kind: knowledge
status: current
created_at: 2026-05-22T19:40:00+09:00
tags: [aws, efs, operations, access-point, iam, posix]
home_hub: efs
---

# EFS Access Points + IAM

> Access Point = sub-directory + uid/gid 강제. IAM = 누가 mount 가능 + 어떤 action.

## 1. Access Point 개념

```
EFS Filesystem (root /)
├─ Access Point bkurs-app (subDirectory=/app, uid=1001, gid=1001)
│   └─ /app/files (사용자 보기엔 / 처럼)
├─ Access Point ml-models (subDirectory=/models, uid=2000, gid=2000)
│   └─ /models/...
```

- 하나의 FS 안 multi-tenant
- 각 tenant 가 자체 root 처럼
- uid/gid 강제 (POSIX permission 무시 가능)

## 2. 생성

```bash
aws efs create-access-point --file-system-id fs-xxx \
  --posix-user "Uid=1001,Gid=1001" \
  --root-directory "Path=/app,CreationInfo={OwnerUid=1001,OwnerGid=1001,Permissions=0755}" \
  --tags Key=Name,Value=bkurs-app
```

| 옵션 | 의미 |
| --- | --- |
| `posix-user` | mount 시 강제 uid/gid (실제 client uid 무시) |
| `root-directory.Path` | filesystem 안의 sub-directory |
| `root-directory.CreationInfo` | path 없을 시 자동 생성 + permission |
| `client-token` | idempotency |

## 3. Mount with Access Point

```bash
sudo mount -t efs -o tls,accesspoint=fsap-xxx \
  fs-xxx:/ /mnt/app

# 실제 mount 후
df -h /mnt/app
# fs-xxx:/  ...   /mnt/app

# uid 1001 로 file 생성
touch /mnt/app/new.txt
ls -la /mnt/app/new.txt
# -rw-r--r-- 1 1001 1001 0 May 22 19:00 new.txt
```

## 4. EKS Pod 의 Access Point 활용

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: efs-app }
spec:
  capacity: { storage: 5Gi }
  volumeMode: Filesystem
  accessModes: [ReadWriteMany]
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxx::fsap-bkurs-app   # FS + Access Point
```

→ Pod 가 sub-directory + uid 1001 로만 접근.

## 5. ECS Task 의 EFS

```json
{
  "volumes": [{
    "name": "efs-app",
    "efsVolumeConfiguration": {
      "fileSystemId": "fs-xxx",
      "rootDirectory": "/",
      "transitEncryption": "ENABLED",
      "authorizationConfig": {
        "accessPointId": "fsap-bkurs-app",
        "iam": "ENABLED"
      }
    }
  }],
  "containerDefinitions": [{
    "mountPoints": [{ "sourceVolume": "efs-app", "containerPath": "/data" }]
  }]
}
```

## 6. IAM Authorization (별도, POSIX 외)

```bash
aws efs put-file-system-policy --file-system-id fs-xxx \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "RequireTransitEncryption",
        "Effect": "Deny",
        "Principal": { "AWS": "*" },
        "Action": "*",
        "Resource": "arn:aws:elasticfilesystem:ap-northeast-2:ACCT:file-system/fs-xxx",
        "Condition": {
          "Bool": { "aws:SecureTransport": "false" }
        }
      },
      {
        "Sid": "AllowSpecificRole",
        "Effect": "Allow",
        "Principal": { "AWS": "arn:aws:iam::ACCT:role/bkurs-app" },
        "Action": [
          "elasticfilesystem:ClientMount",
          "elasticfilesystem:ClientWrite"
        ],
        "Resource": "arn:aws:elasticfilesystem:ap-northeast-2:ACCT:file-system/fs-xxx"
      }
    ]
  }'
```

| Action | 의미 |
| --- | --- |
| `elasticfilesystem:ClientMount` | mount 허용 |
| `elasticfilesystem:ClientRead` | read |
| `elasticfilesystem:ClientWrite` | write |
| `elasticfilesystem:ClientRootAccess` | root 권한 (uid 0) |

## 7. IAM Mount Helper

```bash
sudo mount -t efs -o tls,iam,accesspoint=fsap-xxx \
  fs-xxx:/ /mnt/app
```

→ EC2 의 IAM Instance Profile / ECS Task Role 사용. POSIX 외 추가 layer.

## 8. POSIX vs IAM 결합

| | POSIX | IAM | Access Point |
| --- | --- | --- | --- |
| Mount 시 인증 | NFS (anyone) | IAM credential | uid/gid 강제 |
| File-level permission | uid/gid + mode | n/a | uid/gid 강제 |
| Action 별 권한 | rwx | Mount/Read/Write/Root | n/a |
| Multi-tenant | 어려움 | 가능 | ★ 권장 |

→ **Access Point + IAM + POSIX 결합 권장**.

## 9. Access Point + ECS Task Role 시나리오

```
1. ECS Task 가 ECS Task Role 의 IAM 사용
2. EFS Filesystem Policy 가 Task Role 에 elasticfilesystem:ClientMount/Write 허용
3. Access Point 가 sub-directory + uid 1001 강제
4. Container 안에서 /data 로 mount → /app/files 만 보임
5. 같은 FS 의 다른 Access Point (다른 service) 와 완전 격리
```

## 10. Cross-account Access Point

```bash
# EFS Filesystem Policy 에 다른 account 허용
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::OTHER_ACCT:role/Caller" },
  "Action": "elasticfilesystem:ClientMount",
  "Resource": "..."
}
```

> EFS 의 Mount Target 이 같은 VPC 안 — Cross-VPC = PrivateLink / Transit Gateway.

## 11. Access Point 의 한계

| 한계 | |
| --- | --- |
| FS 당 1000 max | 더 필요 시 새 FS |
| uid/gid 강제 = client 의 실제 uid 무시 | 의도된 동작 |
| sub-directory 변경 불가 | 새 AP 생성 |
| Mount Target 의 SG 그대로 | 따로 |

## 12. Lambda + EFS Access Point

```bash
aws lambda update-function-configuration --function-name bkurs-ml \
  --file-system-configs Arn=arn:efs:...:access-point/fsap-ml-models,LocalMountPath=/mnt/models \
  --vpc-config "SubnetIds=...,SecurityGroupIds=..."
```

```python
# Lambda 안에서
import os
model_path = "/mnt/models/large-model.bin"
```

→ 대용량 ML model / dependency 를 EFS 에서 (Lambda 250 MB limit 우회).

## 13. CloudTrail audit

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateAccessPoint \
  --max-items 20
```

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Access Point 없이 multi-tenant | per-tenant AP |
| posix-user 없으면 root mount | 명시 |
| IAM authorization 안 함 | Filesystem Policy |
| Transit Encryption 미강제 | Deny SecureTransport=false |
| Lambda + EFS NAT 비용 | VPC Endpoint 안 됨 (EFS 자체가 VPC) |
| Cross-account Mount Target 없음 | 같은 VPC 또는 PrivateLink |

## 15. 다음

- ▶ [[02-backup-and-replication]]
- ▶ [[03-mount-and-csi]]
- ▶ [[../practice/01-mount-ec2-eks]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [EFS Access Points](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html) |
| [EFS IAM](https://docs.aws.amazon.com/efs/latest/ug/iam-access-control-nfs-efs.html) |

## 17. 참고

- [[../efs]]
- [[../../iam/iam]]
- [[../../compute/eks/02-workload/03-storage-csi]]
