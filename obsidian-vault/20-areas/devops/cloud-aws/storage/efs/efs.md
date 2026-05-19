---
title: "EFS — Elastic File System (NFS)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:10:00+09:00
tags: [devops, cloud-aws, efs, nfs]
---

# EFS — Elastic File System (NFS)

**[[cloud-aws|↑ cloud-aws]]**

---

## 1. 무엇

```
AWS managed NFSv4:
  - 여러 EC2 / ECS / EKS / Lambda 가 동시 mount
  - 자동 확장 (PB+)
  - multi-AZ 자동 replication
  - encryption at rest + in transit
  
vs EBS:
  EBS: 한 EC2 에 attach (block)
  EFS: 여러 EC2 가 동시 mount (file, NFS)
  
vs S3:
  S3: object (HTTP API)
  EFS: file (NFS, POSIX)
```

---

## 2. 비용

```
Standard:           $0.30/GB-mo
Infrequent Access:  $0.025/GB-mo (★)
One Zone:           $0.16/GB-mo (★ single AZ)
One Zone-IA:        $0.0133/GB-mo

Throughput:
  Bursting (default): 자동, baseline / burst credit
  Provisioned: 명시 MB/s 보장 ($)
  Elastic: 자동 scale (★ 권장)

vs S3:           $0.023/GB-mo
vs EBS gp3:      $0.08/GB-mo
```

→ EFS 가 EBS 보다 비쌈. 단 multi-mount + 자동 scale.

---

## 3. 사용 사례

```
✓ 적합:
  - WordPress / CMS (여러 web server 가 공유 file)
  - 개발 share (homedir, build cache)
  - container 의 RWX volume
  - SageMaker training data
  - Lambda 의 large dependency (NodeJS modules)
  - 일반 NFS replace

✗ 부적합:
  - 매우 작은 file 많음 (latency)
  - 매우 빠른 IO (DB)
  - 1 server 만 사용 (EBS)
  - object storage 가 더 cheap (S3)
```

---

## 4. 생성 (Terraform)

```hcl
resource "aws_efs_file_system" "main" {
  creation_token = "main-efs"
  
  performance_mode = "generalPurpose"     # 또는 maxIO (high IO)
  throughput_mode  = "elastic"             # ★ 권장
  
  encrypted      = true
  kms_key_id     = aws_kms_key.efs.arn
  
  lifecycle_policy {
    transition_to_ia                    = "AFTER_30_DAYS"
    transition_to_primary_storage_class = "AFTER_1_ACCESS"
  }
  
  tags = {Name = "main-efs"}
}

# multi-AZ mount target
resource "aws_efs_mount_target" "main" {
  count           = length(var.private_subnet_ids)
  file_system_id  = aws_efs_file_system.main.id
  subnet_id       = var.private_subnet_ids[count.index]
  security_groups = [aws_security_group.efs.id]
}

resource "aws_security_group" "efs" {
  vpc_id = var.vpc_id
  
  ingress {
    from_port       = 2049         # NFS port
    to_port         = 2049
    protocol        = "tcp"
    security_groups = [var.app_sg_id]
  }
}
```

---

## 5. EC2 mount

```bash
# package
sudo yum install -y amazon-efs-utils    # Amazon Linux
sudo apt install -y nfs-common stunnel4 # Ubuntu

# mount (EFS helper, TLS)
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls fs-0abc123:/ /mnt/efs

# /etc/fstab 영구
fs-0abc123:/    /mnt/efs    efs    defaults,_netdev,tls,iam    0 0
```

→ `tls` = encryption in transit. `iam` = IAM authentication.

---

## 6. IAM authentication (★)

```hcl
# Identity-based policy on IAM role
resource "aws_iam_policy" "efs_access" {
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite",
        "elasticfilesystem:ClientRootAccess"
      ]
      Resource = aws_efs_file_system.main.arn
      Condition = {
        Bool = {"elasticfilesystem:AccessedViaMountTarget" = "true"}
      }
    }]
  })
}
```

→ EBS / S3 처럼 IAM 의 access. 없이 mount 시 fail.

---

## 7. Access Point (★ 권장)

```hcl
# 사용자 별 / app 별 가상 root
resource "aws_efs_access_point" "app" {
  file_system_id = aws_efs_file_system.main.id
  
  root_directory {
    path = "/app-data"
    creation_info {
      owner_uid   = 1000
      owner_gid   = 1000
      permissions = "755"
    }
  }
  
  posix_user {
    uid = 1000
    gid = 1000
  }
  
  tags = {Name = "app-ap"}
}
```

```bash
# mount via access point
sudo mount -t efs -o tls,accesspoint=fsap-0abc fs-0abc:/ /mnt/efs

# → /app-data 만 보임 (root 권한 X)
# → 모든 file 의 uid = 1000 (강제)
```

→ multi-tenant 환경에서 격리.

---

## 8. EKS 통합 (CSI driver)

```bash
# EFS CSI driver 설치 (★)
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    -n kube-system \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
```

```yaml
# StorageClass (dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: efs-sc}
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0abc123
  directoryPerms: "700"
  uid: "1000"
  gid: "1000"
  basePath: "/dynamic_provisioning"

---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: shared-data, namespace: app}
spec:
  accessModes: [ReadWriteMany]      # ★ RWX
  storageClassName: efs-sc
  resources: {requests: {storage: 5Gi}}    # EFS = 무시되지만 명시

---
# Pod 사용
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: web
          image: nginx:alpine
          volumeMounts:
            - {mountPath: /usr/share/nginx/html, name: data}
      volumes:
        - name: data
          persistentVolumeClaim: {claimName: shared-data}
```

→ 3 pod 가 같은 EFS 공유. EBS 는 RWO 만.

---

## 9. Lambda 통합

```bash
# Lambda 가 EFS mount → 큰 dependency / large file 처리
aws lambda create-function \
    --function-name large-app \
    --runtime nodejs20.x \
    --vpc-config SubnetIds=...,SecurityGroupIds=... \
    --file-system-configs Arn=arn:aws:elasticfilesystem:...,LocalMountPath=/mnt/efs
```

```javascript
// Lambda 코드
const fs = require('fs');
const data = fs.readFileSync('/mnt/efs/model.bin');
// ...
```

→ Lambda 의 250MB deployment limit 회피.

→ ML model / large dataset / shared cache.

---

## 10. performance mode

```
generalPurpose (default):
  - 7,000 ops/s max
  - 낮은 latency
  - 대부분 workload

maxIO:
  - 더 높은 throughput
  - 더 큰 latency (각 op 의)
  - 큰 batch / 분석 workload
```

```
throughput mode:
  Bursting (default):
    - baseline: 50 KB/s per GB
    - burst: credit 따라 spike 가능
    - 작은 EFS = burst 제한
    
  Provisioned:
    - 명시 MB/s (예: 100 MB/s)
    - 비쌈
    
  Elastic (★ 권장):
    - 자동 scale
    - usage 만큼 과금
    - 대부분 OK
```

---

## 11. lifecycle (IA)

```hcl
lifecycle_policy {
    transition_to_ia                    = "AFTER_30_DAYS"
    transition_to_primary_storage_class = "AFTER_1_ACCESS"
}
```

```
30일 안 쓰면 → IA storage class
access 시 → 다시 Standard

비용 절감 큼:
  $0.30/GB → $0.025/GB (90% saving)

단:
  - first access 시 latency 약간 ↑
  - 자주 access 면 transition cost ↑
```

---

## 12. backup

```
AWS Backup 통합:
  - 자동 daily / weekly snapshot
  - cross-region copy
  - point-in-time restore
```

```hcl
resource "aws_backup_plan" "efs" {
  name = "efs-backup-plan"
  
  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 * * ? *)"        # 매일 5시
    
    lifecycle {
      cold_storage_after = 30
      delete_after        = 365
    }
    
    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn   # cross-region
    }
  }
}
```

---

## 13. monitoring

```
CloudWatch metric:
  - PercentIOLimit (peak 사용률)
  - PermittedThroughput (capacity)
  - MeteredIOBytes
  - StorageBytes (size)
  - BurstCreditBalance (★ 0 임박 = warning)
  - ClientConnections

alert:
  - BurstCreditBalance < 50% → throughput mode 변경 검토
  - StorageBytes 폭증
```

---

## 14. troubleshoot

```bash
# mount 실패
mount -v -t efs -o tls fs-xxx:/ /mnt/efs
# verbose 로 원인 확인

# 흔한 원인:
# 1. SG: NFS port 2049 안 열림
# 2. mount target 없음 (해당 AZ)
# 3. DNS resolution (private DNS hostname 활성)
# 4. IAM 권한 없음 (iam mount option 시)

# performance issue
aws cloudwatch get-metric-data \
    --metric-data-queries '[{
        "Id": "burst",
        "MetricStat": {
            "Metric": {
                "Namespace": "AWS/EFS",
                "MetricName": "BurstCreditBalance",
                "Dimensions": [{"Name": "FileSystemId", "Value": "fs-xxx"}]
            },
            "Period": 60,
            "Stat": "Average"
        }
    }]'

# CloudWatch logs (EFS audit)
aws efs put-account-preferences --resource-id-type LONG_ID
```

---

## 15. EFS vs FSx vs S3

```
EFS:
  Linux NFS
  POSIX
  multi-mount
  
FSx for Windows:
  SMB / Active Directory
  Windows 만
  
FSx for Lustre:
  high-performance (HPC, ML)
  S3 통합 (data repository)
  
FSx for NetApp ONTAP:
  enterprise (snapshot / dedup / FlexClone)
  
FSx for OpenZFS:
  ZFS
  
S3:
  object (HTTP API)
  더 cheap
  POSIX 아님
```

→ general NFS = EFS. ML / HPC = FSx Lustre. Windows = FSx Windows.

---

## 16. 함정

1. **single AZ 사용** — One Zone 의 cheaper 지만 AZ fail 시 unavailable.
2. **SG 2049 port** — 안 열려서 mount fail.
3. **Bursting mode + 큰 throughput** — credit 0 → 느림.
4. **모든 namespace 의 RWX** — 격리 X. Access Point + uid.
5. **lifecycle 30일 IA + 자주 access** — transition cost ↑.
6. **DNS resolution** — VPC 의 `enableDnsHostnames` 필요.
7. **backup 없음** — 실수로 file 삭제.
8. **encryption 안 함** — at rest / in transit 둘 다.

---

## 17. 관련

- [[cloud-aws|↑ cloud-aws]]
- [[s3]]
- [[ebs]]
- [[eks|↗ EKS]]
- [[lambda|↗ Lambda]]
