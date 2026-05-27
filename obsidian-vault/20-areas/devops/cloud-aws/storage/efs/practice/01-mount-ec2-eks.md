---
title: "EFS 실습 1 — EC2 + EKS Pod 동시 mount + Access Point"
kind: practice
status: current
created_at: 2026-05-22T19:46:00+09:00
tags: [aws, efs, practice, ec2, eks, access-point, csi]
home_hub: efs
---

# 실습 1 — EFS + EC2 + EKS 동시 mount

> 목표: EFS Standard Elastic → EC2 mount + EKS Pod CSI mount + 동시 write 확인. 45분.

> EKS 의 lab-eks (실습) cluster 있다고 가정.

## 1. KMS Key + EFS Filesystem

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

# KMS
KEY_ID=$(aws kms create-key --description "lab efs" \
  --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/lab-efs --target-key-id $KEY_ID

# EFS
FS_ID=$(aws efs create-file-system \
  --creation-token lab-shared-$(date +%s) \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted --kms-key-id alias/lab-efs \
  --backup \
  --tags Key=Name,Value=lab-shared Key=Backup,Value=daily \
  --query 'FileSystemId' --output text)

echo "FS_ID=$FS_ID"
```

## 2. Mount Target (per AZ)

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)

# Client SG
SG_CLIENT=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-efs-client \
  --description "efs client" --query GroupId --output text)

# EFS SG (2049 from client)
SG_EFS=$(aws ec2 create-security-group --vpc-id $VPC --group-name lab-efs \
  --description "efs" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_EFS \
  --protocol tcp --port 2049 --source-group $SG_CLIENT

# Mount Target per AZ
for AZ in a b c; do
  SUBNET=$(aws ec2 describe-subnets \
    --filters Name=availability-zone,Values=ap-northeast-2$AZ Name=vpc-id,Values=$VPC \
    --query 'Subnets[0].SubnetId' --output text)
  aws efs create-mount-target --file-system-id $FS_ID \
    --subnet-id $SUBNET --security-groups $SG_EFS
done

aws efs describe-mount-targets --file-system-id $FS_ID \
  --query 'MountTargets[].{AZ:AvailabilityZoneName,IP:IpAddress,State:LifeCycleState}'
```

## 3. Access Point — multi-tenant

```bash
# app AP
AP_APP=$(aws efs create-access-point --file-system-id $FS_ID \
  --posix-user "Uid=1001,Gid=1001" \
  --root-directory "Path=/app,CreationInfo={OwnerUid=1001,OwnerGid=1001,Permissions=0755}" \
  --tags Key=Name,Value=app \
  --query 'AccessPointId' --output text)

# ml AP
AP_ML=$(aws efs create-access-point --file-system-id $FS_ID \
  --posix-user "Uid=2000,Gid=2000" \
  --root-directory "Path=/ml,CreationInfo={OwnerUid=2000,OwnerGid=2000,Permissions=0755}" \
  --tags Key=Name,Value=ml \
  --query 'AccessPointId' --output text)
```

## 4. Filesystem Policy — IAM + TLS 강제

```bash
aws efs put-file-system-policy --file-system-id $FS_ID --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNoTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "Resource": "arn:aws:elasticfilesystem:ap-northeast-2:$ACCT:file-system/$FS_ID",
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
EOF
)"
```

## 5. EC2 Mount

```bash
AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)
SUBNET=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC \
  Name=defaultForAz,Values=true --query 'Subnets[0].SubnetId' --output text)

INST=$(aws ec2 run-instances --image-id $AMI --instance-type t4g.small \
  --subnet-id $SUBNET --security-group-ids $SG_CLIENT \
  --iam-instance-profile Name=FirstEc2Profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lab-efs-ec2}]' \
  --query 'Instances[0].InstanceId' --output text)

sleep 60

# SSM Mount
aws ssm send-command --instance-ids $INST \
  --document-name AWS-RunShellScript \
  --parameters "commands=[
    \"dnf install -y amazon-efs-utils\",
    \"mkdir -p /mnt/app\",
    \"mount -t efs -o tls,accesspoint=$AP_APP $FS_ID:/ /mnt/app\",
    \"echo 'from ec2' | sudo -u '#1001' tee /mnt/app/ec2.txt\",
    \"ls -la /mnt/app/\"
  ]"
```

## 6. EKS Pod Mount (EFS CSI)

```bash
# IRSA for EFS CSI
eksctl create iamserviceaccount --cluster lab-eks \
  --namespace kube-system --name efs-csi-controller-sa \
  --role-name LabEFSCSIRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --override-existing-serviceaccounts --approve

# Add-on
aws eks create-addon --cluster-name lab-eks --addon-name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::$ACCT:role/LabEFSCSIRole
```

```yaml
# StorageClass (dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: efs-sc }
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: $FS_ID
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic"
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: shared, namespace: default }
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
  resources: { requests: { storage: 5Gi } }
---
# Pod (write)
apiVersion: v1
kind: Pod
metadata: { name: writer, namespace: default }
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh","-c","while true; do echo from-eks-$(hostname) >> /shared/eks.txt; sleep 5; done"]
      volumeMounts: [{ name: shared, mountPath: /shared }]
  volumes:
    - name: shared
      persistentVolumeClaim: { claimName: shared }
```

```bash
kubectl apply -f efs-pod.yaml
sleep 30
kubectl exec writer -- cat /shared/eks.txt
```

## 7. Static PV with Existing Access Point

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: efs-app-static }
spec:
  capacity: { storage: 5Gi }
  volumeMode: Filesystem
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  csi:
    driver: efs.csi.aws.com
    volumeHandle: $FS_ID::$AP_APP
```

## 8. ECS Fargate + EFS

```bash
# Task definition
cat > taskdef.json <<EOF
{
  "family": "lab-efs-ecs",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256", "memory": "512",
  "runtimePlatform": { "cpuArchitecture": "ARM64", "operatingSystemFamily": "LINUX" },
  "executionRoleArn": "arn:aws:iam::$ACCT:role/ecsTaskExecutionRole",
  "volumes": [{
    "name": "efs-shared",
    "efsVolumeConfiguration": {
      "fileSystemId": "$FS_ID",
      "rootDirectory": "/app",
      "transitEncryption": "ENABLED",
      "authorizationConfig": {
        "accessPointId": "$AP_APP"
      }
    }
  }],
  "containerDefinitions": [{
    "name": "app",
    "image": "public.ecr.aws/docker/library/busybox:latest",
    "essential": true,
    "command": ["sh","-c","while true; do echo from-ecs-\\$(hostname) >> /shared/ecs.txt; sleep 5; done"],
    "mountPoints": [{ "sourceVolume": "efs-shared", "containerPath": "/shared" }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-create-group": "true",
        "awslogs-group": "/ecs/lab-efs",
        "awslogs-region": "ap-northeast-2",
        "awslogs-stream-prefix": "app"
      }
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://taskdef.json
```

## 9. Concurrent Write 검증

```bash
# EC2 의 /mnt/app/ec2.txt
# EKS 의 /shared/eks.txt
# ECS 의 /shared/ecs.txt

# 같은 Access Point (Path=/app) 이지만 같은 file 의 동시 write 도 OK
aws ssm send-command --instance-ids $INST \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["sudo ls /mnt/app/"]'

# EKS
kubectl exec writer -- ls /shared/
```

## 10. Backup 자동

```bash
# tag 으로 활성
aws efs tag-resource --resource-id $FS_ID --tags Key=Backup,Value=daily

# 자동 backup 활성 확인
aws efs describe-backup-policy --file-system-id $FS_ID
```

→ Backup=daily tag 가 있는 EFS 가 AWS Backup plan 에 자동 포함.

## 11. CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name ClientConnections \
  --dimensions Name=FileSystemId,Value=$FS_ID \
  --period 60 --statistics Maximum --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

## 12. Clean up

```bash
kubectl delete pod writer
kubectl delete pvc shared
kubectl delete storageclass efs-sc

aws ec2 terminate-instances --instance-ids $INST

aws efs delete-access-point --access-point-id $AP_APP
aws efs delete-access-point --access-point-id $AP_ML

# Mount Targets
for MT in $(aws efs describe-mount-targets --file-system-id $FS_ID --query 'MountTargets[].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id $MT
done
sleep 30

aws efs delete-file-system --file-system-id $FS_ID

aws ec2 delete-security-group --group-id $SG_EFS
aws ec2 delete-security-group --group-id $SG_CLIENT
aws kms schedule-key-deletion --key-id $KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/lab-efs
```

## 13. 학습 포인트

- amazon-efs-utils + TLS mount
- Mount Target per-AZ
- Access Point 의 uid/gid 강제 + sub-directory
- Filesystem Policy 의 TLS 강제
- EKS CSI Dynamic Provisioning (Access Point 자동 생성)
- ECS Fargate + EFS (efsVolumeConfiguration)
- EC2 + EKS + ECS 동시 write 의 lock-free coexistence

## 14. 다음

- ▶ [[02-lifecycle-and-replication]]
- ▶ [[../02-operations/01-access-points-and-iam]]
- ▶ [[../../../compute/eks/02-workload/03-storage-csi]]

## 15. 참고

- [[../efs]]
- [[../02-operations/03-mount-and-csi]]
