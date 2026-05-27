---
title: "ECR 02-2 — Lifecycle Policy + Cross-Region Replication"
kind: knowledge
status: current
created_at: 2026-05-22T17:44:00+09:00
tags: [aws, ecr, operations, lifecycle, replication, dr]
home_hub: ecr
---

# ECR Lifecycle + Replication

> Lifecycle Policy = 자동 expire / Cross-Region Replication = DR mirror.

## 1. Lifecycle Policy

```bash
aws ecr put-lifecycle-policy --repository-name bkurs-be \
  --lifecycle-policy-text "$(cat <<'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 30 production images (prod- prefix)",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod-"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 10 staging images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["staging-"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "Remove untagged after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
EOF
)"
```

| selection 필드 | 의미 |
| --- | --- |
| `tagStatus` | tagged / untagged / any |
| `tagPrefixList` | prefix 필터 |
| `tagPatternList` | 정규식 패턴 (`v[0-9]+.[0-9]+.[0-9]+`) |
| `countType` | `imageCountMoreThan` 또는 `sinceImagePushed` |
| `countNumber` | 보존 수 또는 일수 |
| `countUnit` | days (sinceImagePushed) |

## 2. Lifecycle 우선순위

```
priority 1, 2, 3 ... 순서대로 평가.
첫 매치 rule 적용, 다른 rule 무시.
→ 가장 협소한 rule 을 priority 1 로.
```

## 3. Lifecycle 의 dry-run

```bash
aws ecr start-lifecycle-policy-preview --repository-name bkurs-be \
  --lifecycle-policy-text "$(cat policy.json)"

aws ecr get-lifecycle-policy-preview --repository-name bkurs-be \
  --query 'previewResults[].imageIds'
```

→ 실제 삭제 안 하고 결과 확인.

## 4. Lifecycle 실행 주기

- 24시간마다 자동
- 또는 수동 trigger (정책 변경 시 새로)
- 결과: CloudTrail / CloudWatch Events 의 `ECR Image Action` 이벤트

## 5. 보존 우선 patterns

```json
// 최신 N 유지 (production)
{
  "selection": {
    "tagStatus": "tagged",
    "tagPrefixList": ["v"],
    "countType": "imageCountMoreThan",
    "countNumber": 50
  },
  "action": { "type": "expire" }
}

// 30일 후 untagged 삭제
{
  "selection": {
    "tagStatus": "untagged",
    "countType": "sinceImagePushed",
    "countUnit": "days",
    "countNumber": 30
  },
  "action": { "type": "expire" }
}

// PR 빌드 14일 후 삭제
{
  "selection": {
    "tagStatus": "tagged",
    "tagPatternList": ["pr-*"],
    "countType": "sinceImagePushed",
    "countUnit": "days",
    "countNumber": 14
  },
  "action": { "type": "expire" }
}

// development 빌드 1일 retention
{
  "selection": {
    "tagStatus": "tagged",
    "tagPrefixList": ["dev-"],
    "countType": "sinceImagePushed",
    "countUnit": "days",
    "countNumber": 1
  },
  "action": { "type": "expire" }
}
```

## 6. Cross-Region Replication

```bash
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [{
    "destinations": [
      { "region": "ap-northeast-1", "registryId": "123456789012" }
    ],
    "repositoryFilters": [
      { "filter": "bkurs", "filterType": "PREFIX_MATCH" }
    ]
  }]
}'
```

- account 단위 설정
- repository prefix filter
- 다른 region (또는 다른 account)
- one-way only (source → dest)
- async (대기 시간 짧음, image 별 ~수초)

## 7. Cross-Account Replication

```json
{
  "rules": [{
    "destinations": [
      { "region": "ap-northeast-2", "registryId": "987654321098" }
    ]
  }]
}
```

> Source account 에서 push 시 자동으로 destination account 에 mirror.

다만 destination account 의 ECR repository 가 미리 존재해야. 자동 생성 옵션:
```json
{
  "repositoryName": {
    "prefix": "shared-",
    "service": "ecr"
  }
}
```

## 8. Replication 의 비용

```
Source storage:  $0.10/GB·월
Dest storage:    $0.10/GB·월 (별도 부과)
DT cross-region: $0.09/GB (one-time, push 시)
```

= image 1 GB × 2 region = $0.20/월 + $0.09 first push.

## 9. Replication 검증

```bash
# Source
aws ecr list-images --repository-name bkurs-be --region ap-northeast-2

# Dest
aws ecr list-images --repository-name bkurs-be --region ap-northeast-1

# Replication status
aws ecr describe-image-replication-status --repository-name bkurs-be \
  --image-id imageTag=v1.0.0
```

## 10. DR 활용 패턴

```mermaid
flowchart LR
  CI[CI build] -->|push| ECR1[(ECR ap-northeast-2 source)]
  ECR1 -->|replication| ECR2[(ECR ap-northeast-1 DR)]
  ECR1 -->|pull| ECS1[ECS prod]
  ECR2 -->|pull (DR)| ECS2[ECS DR cluster]
```

> Primary region down → DR region 의 ECS 가 같은 image 즉시 사용.

## 11. Lifecycle + Replication 결합

| 시나리오 | |
| --- | --- |
| Production 30 image source | source 에 lifecycle 적용 |
| DR region | replication 으로 동일 30 image (자동) |

→ source 에 expire 발생 시 destination 도 따라감.

## 12. Replication 의 한계

| 한계 | 회피 |
| --- | --- |
| One-way only | 양방향 = 두 account 모두 rule |
| 새 push 만 replicate (기존 image X) | manual copy |
| 30 dest 한도 | 증액 |
| Pull-through cache 는 replication 안 됨 | per-region cache |
| Repository policy 는 replication 안 됨 | dest 에 별도 |

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Lifecycle 없음 → storage 폭주 | 항상 설정 |
| Untagged image 무한 누적 | 7d expire |
| Priority 순서 오류 → 의도와 다른 expire | priority 1 가장 좁은 |
| Replication enable 시 기존 image 안 됨 | manual copy first |
| Cross-account replication 의 repository 자동 생성 옵션 모름 | settings |
| Replication storage 비용 인지 못함 | DR 만 필요 region |

## 14. 다음

- ▶ [[03-cross-account-and-region]]
- ▶ [[../03-security/01-scanning-and-inspector]]
- ▶ [[../03-security/02-immutability-and-signing]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Lifecycle Policy](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html) |
| [Replication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry-replication.html) |

## 16. 참고

- [[../ecr]]
- [[01-push-pull-auth]]
- [[../01-fundamentals/03-pricing-and-limits]]
