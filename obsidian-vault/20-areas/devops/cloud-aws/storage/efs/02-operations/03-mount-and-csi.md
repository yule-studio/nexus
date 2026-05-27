---
title: "EFS 02-3 — Mount Helper + EKS CSI + ECS + Lambda"
kind: knowledge
status: current
created_at: 2026-05-22T19:44:00+09:00
tags: [aws, efs, operations, mount, nfs, csi, eks, ecs, lambda]
home_hub: efs
---

# EFS Mount + EKS CSI + ECS + Lambda

> amazon-efs-utils (mount helper) + EKS CSI Driver + ECS efsVolumeConfiguration + Lambda fileSystemConfig.

## 1. amazon-efs-utils

```bash
# Amazon Linux 2023
sudo dnf install -y amazon-efs-utils

# Ubuntu / Debian (NFS only — 자동 helper 없음)
sudo apt-get install -y nfs-common
```

→ TLS / stunnel 자동 + mount helper.

## 2. Mount (EC2 직접)

```bash
# 기본
sudo mount -t efs fs-xxx:/ /mnt/efs

# TLS
sudo mount -t efs -o tls fs-xxx:/ /mnt/efs

# Access Point
sudo mount -t efs -o tls,accesspoint=fsap-xxx fs-xxx:/ /mnt/efs

# IAM authorization
sudo mount -t efs -o tls,iam,accesspoint=fsap-xxx fs-xxx:/ /mnt/efs

# fstab 등록
echo "fs-xxx:/ /mnt/efs efs _netdev,tls,accesspoint=fsap-xxx 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

## 3. NFS direct (efs-utils 없을 때)

```bash
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  fs-xxx.efs.ap-northeast-2.amazonaws.com:/ /mnt/efs

# IPv4
sudo mount -t nfs4 ... 10.0.x.x:/ /mnt/efs
```

## 4. EKS CSI Driver

```bash
# Managed Add-on
aws eks create-addon --cluster-name bkurs-prod --addon-name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::ACCT:role/EFSCSIRole
```

IAM Role permissions:
```json
{
  "Effect": "Allow",
  "Action": [
    "elasticfilesystem:DescribeAccessPoints",
    "elasticfilesystem:DescribeFileSystems",
    "elasticfilesystem:DescribeMountTargets",
    "ec2:DescribeAvailabilityZones"
  ],
  "Resource": "*"
}
```

### Static PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: efs-pv }
spec:
  capacity: { storage: 5Gi }
  volumeMode: Filesystem
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxx::fsap-bkurs-app
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: efs-pvc, namespace: prod }
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
  resources: { requests: { storage: 5Gi } }
  volumeName: efs-pv
```

### Dynamic Provisioning
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: efs-sc }
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxx
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: shared-files, namespace: prod }
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
  resources: { requests: { storage: 5Gi } }
```

→ PVC 마다 EFS Access Point 자동 생성. Auto-cleanup on delete.

## 5. EKS Pod 의 mount

```yaml
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: bkurs-be:v1
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    - name: shared
      persistentVolumeClaim: { claimName: shared-files }
```

## 6. ECS Task EFS

```json
{
  "family": "bkurs-api",
  "networkMode": "awsvpc",
  "volumes": [{
    "name": "efs-shared",
    "efsVolumeConfiguration": {
      "fileSystemId": "fs-xxx",
      "rootDirectory": "/",
      "transitEncryption": "ENABLED",
      "transitEncryptionPort": 2999,
      "authorizationConfig": {
        "accessPointId": "fsap-bkurs-app",
        "iam": "ENABLED"
      }
    }
  }],
  "containerDefinitions": [{
    "name": "app",
    "mountPoints": [{ "sourceVolume": "efs-shared", "containerPath": "/shared", "readOnly": false }]
  }]
}
```

> Fargate 도 EFS OK.

## 7. Lambda + EFS

```bash
aws lambda update-function-configuration --function-name bkurs-ml \
  --vpc-config "SubnetIds=subnet-priv-a,subnet-priv-b,SecurityGroupIds=sg-lambda-efs" \
  --file-system-configs Arn=arn:aws:elasticfilesystem:...:access-point/fsap-ml-models,LocalMountPath=/mnt/models
```

```python
import os
def lambda_handler(event, context):
    model_path = "/mnt/models/whisper-large.bin"
    model = load_model(model_path)
    ...
```

활용:
- 큰 ML model (250 MB 한도 우회)
- 공통 dependency (multi-Lambda share)
- Persistent state (database 보다 단순)

## 8. SG (NFS 2049)

```bash
SG_EFS=$(aws ec2 create-security-group --vpc-id vpc-xxx --group-name efs-bkurs \
  --description "efs" --query GroupId --output text)

# Inbound 2049 from client SG
aws ec2 authorize-security-group-ingress --group-id $SG_EFS \
  --protocol tcp --port 2049 --source-group sg-clients
```

## 9. Performance Tuning

```bash
# Mount options
-o rsize=1048576,wsize=1048576    # large buffers
-o hard                            # data integrity
-o timeo=600                       # 60s timeout
-o noatime                         # access time 미기록 (성능 ↑)
```

## 10. NFS client 의 cache

```
Linux NFS client 의 attribute cache (default):
- ac=true (default)
- acregmin=3s, acregmax=60s

큰 cache (variation):
-o ac,actimeo=60

No cache (strict consistency):
-o noac
```

> 동시 write 시 strict consistency 필요하면 `noac` (성능 trade-off).

## 11. EKS Fargate 의 EFS

```yaml
# Fargate Profile + EFS 의 PVC
# Fargate task 가 EFS access 가능 — EBS 안 됨 (EFS 만)
```

> Fargate Pod 의 stateful storage = **EFS only**.

## 12. CloudWatch metric

per-client:
```bash
aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name ClientConnections \
  --dimensions Name=FileSystemId,Value=fs-xxx \
  --period 60 --statistics Maximum

aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name TotalIOBytes \
  --dimensions Name=FileSystemId,Value=fs-xxx \
  --period 60 --statistics Sum
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| amazon-efs-utils 없이 NFS direct | tls 어려움 |
| TLS 없이 mount | encryption in transit X |
| SG 2049 차단 | source-group |
| EKS CSI add-on 없음 | enable + IRSA |
| Fargate + EBS | EFS 만 |
| NFS client cache 너무 김 → stale read | actimeo 짧게 또는 noac |
| Lambda + EFS 의 cold start ↑ | warm Lambda 또는 PC |

## 14. 권장 — bkurs-be 통합

```
Filesystem:        Standard, Elastic Throughput, encrypted
Mount Target:      3 AZ (subnet 각각)
Access Point:      service 별 (bkurs-app, ml-models, audit-logs)
Mount option:      tls,iam,accesspoint=xxx,rsize=1048576,wsize=1048576

ECS Task:          efsVolumeConfiguration + accessPointId
EKS Pod:           EFS CSI + Dynamic PV
Lambda:            fileSystemConfig (ML model)
EC2:               efs-utils + fstab
```

## 15. 다음

- ▶ [[../practice/01-mount-ec2-eks]]
- ▶ [[../practice/02-lifecycle-and-replication]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [amazon-efs-utils](https://github.com/aws/efs-utils) |
| [EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver) |
| [Lambda + EFS](https://docs.aws.amazon.com/lambda/latest/dg/configuration-filesystem.html) |

## 17. 참고

- [[../efs]]
- [[01-access-points-and-iam]]
- [[../../../compute/eks/02-workload/03-storage-csi]]
- [[../../../compute/lambda/03-runtime/03-vpc-and-networking]]
