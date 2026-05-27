---
title: "troubleshooting · Terraform state lock"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, terraform, iac, dynamodb, s3]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# Terraform — `Error acquiring the state lock`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
Error: Error acquiring the state lock
Lock Info:
  ID:        9f2e8c1a-3b4d-4f5e-8a7b-1c2d3e4f5a6b
  Path:      myapp-prod/terraform.tfstate
  Operation: OperationTypeApply
  Who:       ci@host
  Created:   2026-05-27 10:00:00 UTC
```

## 원인

이전 `terraform plan/apply` 가 비정상 종료 (Ctrl+C / pod 죽음 / network 끊김) → DynamoDB lock row 가 release 안 됨.

## 해결

### 다른 작업 진행 중인지 확인 (먼저)
```bash
# Lock ID 의 사람 / 시간 확인
# CI log / Slack / 동료 확인

# 진행 중이면 기다림 (몇 분)
```

→ 다른 사람이 정말 apply 중이면 force-unlock 은 위험 (race condition).

### force-unlock
```bash
terraform force-unlock 9f2e8c1a-3b4d-4f5e-8a7b-1c2d3e4f5a6b
# 또는 -force 로 confirmation skip
terraform force-unlock -force 9f2e8c1a-3b4d-4f5e-8a7b-1c2d3e4f5a6b
```

### DynamoDB 직접 (force-unlock 작동 안 할 때)
```bash
aws dynamodb delete-item \
  --table-name terraform-locks \
  --key '{"LockID":{"S":"myapp-prod/terraform.tfstate"}}'
```

### S3 + DynamoDB backend 설정 (권장)
```hcl
terraform {
  backend "s3" {
    bucket         = "myorg-terraform-state"
    key            = "myapp/prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute { name = "LockID"; type = "S" }
}
```

→ S3 가 state 영속화, DynamoDB 가 concurrent apply 차단.

## State drift / corruption

force-unlock 후 state 가 stale 일 수 있음:

```bash
terraform refresh   # AWS 실제 상태와 동기화
terraform plan      # diff 확인

# state 가 corrupt 면 백업에서 복구
aws s3api list-object-versions --bucket myorg-terraform-state --prefix myapp/prod/
aws s3api get-object --bucket ... --key ... --version-id <id> ./terraform.tfstate.bak
```

→ S3 versioning 필수.

## State import (resource 가 cloud 에 있지만 state 에 없음)
```bash
terraform import aws_instance.example i-1234567890abcdef0
```

## 진단

```bash
# Lock 상태
aws dynamodb get-item \
  --table-name terraform-locks \
  --key '{"LockID":{"S":"myapp-prod/terraform.tfstate"}}'

# State file 직접 확인
aws s3 cp s3://myorg-terraform-state/myapp/prod/terraform.tfstate - | jq '.serial,.lineage'

# 진행 중 작업 (Atlantis / Spacelift / TFC)
# → 해당 도구의 lock 화면 확인
```

## 재발 방지

- S3 + DynamoDB backend 표준 (local state 금지 in team)
- S3 versioning + encryption + lifecycle policy
- CI 의 terraform job 에 timeout 명시 (cancel 시 unlock)
- Atlantis / Terraform Cloud / Spacelift 같은 관리 도구 도입

## 관련

- [[devops]]
