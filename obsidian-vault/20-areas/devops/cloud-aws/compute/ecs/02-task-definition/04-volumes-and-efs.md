---
title: "ECS 02-4 — Volumes / EFS / Ephemeral storage"
kind: knowledge
status: current
created_at: 2026-05-22T13:36:00+09:00
tags: [aws, ecs, task-definition, storage, efs]
home_hub: ecs
---

# Task Definition — Volumes / EFS

> ECS task 의 storage 3 종: 임시 (ephemeral) / EFS (영구 공유) / FSx (Lustre / Windows).
> bind mount 는 EC2 launch 만. Fargate 는 EFS 가 유일한 영구 option.

## 1. storage 종류

| 종류 | 영구성 | Fargate | EC2 | 용도 |
| --- | --- | --- | --- | --- |
| **Ephemeral storage** | task 수명 | ✓ (20~200GB) | ✓ (호스트 EBS) | 임시 work file |
| **bind mount** | host 의 path | ✗ | ✓ | dev / config |
| **EFS** | 영구, 공유 | ✓ | ✓ | DB 외 영구 데이터 / share |
| **FSx for Lustre** | 영구, HPC | ✗ | ✓ | ML / 대용량 read |
| **FSx for Windows** | 영구 | ✗ | ✓ (Windows) | SMB share |
| **Docker volume** | EC2 호스트 | ✗ | ✓ | legacy |

> DB 는 RDS / DynamoDB 권장 — task 안 EFS 위에 DB 띄우지 말 것.

## 2. Ephemeral storage (Fargate)

```json
{
  "ephemeralStorage": { "sizeInGiB": 100 }    // 21~200, default 20
}
```

container 안에서 모든 경로가 이 storage 사용. 자세히 → [[../01-fundamentals/03-fargate-deep#5]]

## 3. EFS — 영구 공유 storage

여러 task 가 동시 read/write — POSIX 호환 NFS.

### 3.1 EFS 생성

```bash
aws efs create-file-system \
  --creation-token bkurs-shared \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted

# Mount Target (subnet 마다)
aws efs create-mount-target \
  --file-system-id fs-xxx \
  --subnet-id subnet-aaa \
  --security-groups sg-efs
```

SG `sg-efs` Inbound 2049 (NFS) from ECS task SG.

### 3.2 Access Point — 권장

EFS root 직접 사용 보다 access point — 권한 / quota / root squash:

```bash
aws efs create-access-point \
  --file-system-id fs-xxx \
  --posix-user "Uid=1000,Gid=1000" \
  --root-directory "Path=/bkurs/uploads,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=0755}"
```

### 3.3 Task Definition — mount

```json
{
  "volumes": [
    {
      "name": "shared-uploads",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-xxx",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "fsap-yyy",
          "iam": "ENABLED"
        }
      }
    }
  ],
  "containerDefinitions": [{
    "name": "api",
    "mountPoints": [
      { "sourceVolume": "shared-uploads", "containerPath": "/var/uploads", "readOnly": false }
    ]
  }]
}
```

### 3.4 Task IAM — EFS 접근

`taskRoleArn` 에 추가:
```json
{
  "Effect": "Allow",
  "Action": [
    "elasticfilesystem:ClientMount",
    "elasticfilesystem:ClientWrite",
    "elasticfilesystem:ClientRootAccess"
  ],
  "Resource": "arn:aws:elasticfilesystem:...:file-system/fs-xxx",
  "Condition": {
    "StringEquals": {
      "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:...:access-point/fsap-yyy"
    }
  }
}
```

## 4. EFS performance / throughput

| Performance mode | 사용 |
| --- | --- |
| **generalPurpose** (default) | 일반 — < 7000 IOPS |
| **maxIO** | 수백+ 동시 클라이언트 (latency 높음) |

| Throughput mode | 사용 |
| --- | --- |
| **bursting** (default) | size 따라 자동 |
| **provisioned** | 명시적 MB/s |
| **elastic** | auto-scale, pay per request |

> 신규: **elastic** 권장 (예측 어려운 워크로드).

## 5. bind mount (EC2 only)

```json
{
  "volumes": [{ "name": "host-data", "host": { "sourcePath": "/data" } }],
  "containerDefinitions": [{
    "mountPoints": [{ "sourceVolume": "host-data", "containerPath": "/app/data" }]
  }]
}
```

> Fargate 는 host 가 microVM — sourcePath 의미 없음. 활용 제한적.

## 6. tmpfs (EC2 only)

```json
{
  "volumes": [{ "name": "tmp", "host": {} }],
  "containerDefinitions": [{
    "linuxParameters": {
      "tmpfs": [
        { "containerPath": "/tmp", "size": 100, "mountOptions": ["noexec"] }
      ]
    }
  }]
}
```

RAM-backed temp — Fargate 미지원.

## 7. 활용 패턴

### 7.1 file upload 영구화
- bkurs-be 의 첨부 file → EFS `/uploads` mount
- 또는 S3 직접 (presigned URL, 권장)

### 7.2 shared cache
- 여러 task 가 같은 EFS 의 cache 공유

### 7.3 ML model 공유
- FSx for Lustre (Fargate 미지원 — EC2 만)

### 7.4 자체 DB
- 권장 안 함 — RDS / DynamoDB 사용

## 8. S3 vs EFS

| 항목 | S3 | EFS |
| --- | --- | --- |
| 인터페이스 | HTTP / SDK | POSIX (NFS) |
| latency | 30~100ms | < 5ms |
| 가격 (1GB/월) | $0.023 | $0.30 |
| 동시 read/write | 무제한 | 제한 |
| 단일 file size | 5TB | 47.9TB |
| ECS mount | ✗ (앱이 SDK 호출) | ✓ (volume mount) |

> 일반 file 저장 — S3 권장 (싸고 scalable). file system 처럼 사용 필요 시 EFS.

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| EFS mount 실패 — SG 2049 막힘 | sg-efs Inbound from sg-task |
| Fargate 의 bind mount | 무의미 — EFS 사용 |
| EFS 비용 폭주 | retention policy / lifecycle / IA (Infrequent Access) |
| access point 없이 root mount | 권한 / 격리 어려움 — access point 권장 |
| IAM 조건 누락 | accessPointArn 조건 명시 |

## 10. 다음

- ▶ [[05-environment-and-secrets]]
- ▶ [[06-iam-roles]] — EFS 접근 권한
- ▶ [[../../storage/s3/s3]] — S3 비교
- ▶ [[../../storage/efs/efs]] — EFS 일반

## 11. 영상 / 자료

| 자료 |
| --- |
| [ECS + EFS — AWS](https://www.youtube.com/watch?v=clKa_3eK9D0) |
| [EFS Best Practices — re:Invent](https://www.youtube.com/watch?v=DiZj5kVjJ8s) |
| 공식: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/efs-volumes.html |

## 12. 참고

- [[../ecs]]
- [[../01-fundamentals/03-fargate-deep]] — ephemeral storage
