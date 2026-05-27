---
title: "ECR 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-22T17:38:00+09:00
tags: [aws, ecr, fundamentals, pricing, limits]
home_hub: ecr
---

# ECR Pricing + Limits

> Storage + Data Transfer + Scan + Replication. 일반 소규모 = ~$5/월.

## 1. 비용 구성

| 항목 | Private | Public |
| --- | --- | --- |
| Storage | $0.10/GB·월 | 무료 50GB / authenticated; $0.09/GB·월 over |
| Replication (dest storage) | $0.10/GB·월 (추가) | n/a |
| Data Transfer (same region) | 무료 | n/a |
| Data Transfer (cross-region) | $0.09/GB (source) | n/a |
| Data Transfer (internet) | $0.09/GB | 5TB 무료 / 월 (월별 누적) |
| Basic Scan | 무료 | 무료 |
| Enhanced Scan (Inspector v2) | $0.09/scan + $0.01/re-scan | 같음 |
| Pull-through Cache | $0.10/GB·월 (cache 만) | n/a |

## 2. 무료 tier

```
Private:  500 MB storage / 월 (12개월, 신규 계정만)
Public:   50 GB anonymous outbound / 월 + storage 50 GB free
```

## 3. 비용 시뮬레이션 — bkurs-be

```
production image 평균 200 MB
30 image (lifecycle keep)
= 6 GB storage × $0.10 = $0.60

dev image 평균 200 MB
20 image
= 4 GB storage × $0.10 = $0.40

Replication ap-northeast-2 → ap-northeast-1:
6 GB × $0.10 (dest) + DT (one-time) = ~$0.60

Pull-through Cache:
2 GB cached × $0.10 = $0.20

Enhanced Scan:
50 image × $0.09 = $4.50 (월별 push 마다 + 지속 re-scan)

총: ~$6/월
```

## 4. 비용 vs Docker Hub

| | Docker Hub Free | Docker Hub Pro | ECR Private |
| --- | --- | --- | --- |
| Public repo | 무제한 | 무제한 | n/a |
| Private repo | 1 | 5 / unlimited | 무제한 |
| Pull/6h | 100 (anon) | 5000 | 무제한 (IAM) |
| 비용 | $0 | $5/월/user | usage 기반 |
| AWS 통합 | ✗ | ✗ | ★★★ |

## 5. Hard limits

| 항목 | Limit |
| --- | --- |
| Repository / region / account | 10000 (soft) |
| Image / repository | 10000 (soft) |
| Layers / image | 1000 |
| Manifest size | 4 MB |
| Tag length | 300 chars |
| Tag / image | 100 (alias) |
| Repository name | 256 chars |
| Pull-through rules | 100 / account / region |
| Cross-region replication rules | 30 |
| Lifecycle policy rules | 50 / repository |
| Repository policy size | 10 KB |

## 6. Service Quotas — 증액

```bash
aws service-quotas request-service-quota-increase \
  --service-code ecr \
  --quota-code L-CFEB8E8D \
  --desired-value 50000  # 50K repository
```

## 7. Free Tier 활용

```
12개월 신규 계정:
- 500 MB private storage 무료
- 50 GB public outbound 무료
- Basic scan 무료
- Pull-through cache 의 cache storage 도 같은 free tier
```

## 8. 비용 절감

```
1. Lifecycle Policy — old image 자동 삭제
2. Multi-arch 1 manifest 로 합치기 (layer 공유)
3. Multi-stage Dockerfile — final image 작게
4. distroless / chainguard base
5. .dockerignore (build context 줄임)
6. Pull-through 의 lifecycle 도 설정
7. Replication 필요할 때만 (dev/staging 미설정)
8. Enhanced Scan 의 비용 모니터 (re-scan 가격 인지)
```

## 9. Cost Explorer

```
Service: Elastic Container Registry
- usage_type:
  - StorageBytes:           private storage
  - DataTransfer-Out-Bytes: outbound
  - ScanRequests:           Enhanced scan
```

## 10. ECR Storage Class

```bash
# Pull-through caching 의 lifecycle 자동
aws ecr put-lifecycle-policy --repository-name dockerhub \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "selection": { "tagStatus": "any", "countType": "sinceImagePushed", "countUnit": "days", "countNumber": 30 },
      "action": { "type": "expire" }
    }]
  }'
```

→ 30일 미사용 cache 자동 삭제.

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| 비용 모니터링 없음 | CloudWatch billing alarm |
| Untagged image 누적 | Lifecycle 7d expire |
| Replication 불필요한 region 까지 | 필요 region 만 |
| Enhanced scan all repo (re-scan 비싸짐) | sensitive 만 |
| Cross-region DT 비용 모름 | 같은 region pull |
| 큰 image (4 GB+) | multi-stage + distroless |

## 12. Cost Allocation Tag

```bash
aws ecr tag-resource --resource-arn arn:aws:ecr:...:repository/bkurs-be \
  --tags Key=Environment,Value=prod Key=Service,Value=api Key=CostCenter,Value=engineering
```

→ Cost Explorer / billing 에서 service / environment 별 비용 visibility.

## 13. 다음

- ▶ [[../02-operations/01-push-pull-auth]]
- ▶ [[../02-operations/02-lifecycle-and-replication]]
- ▶ [[../03-security/01-scanning-and-inspector]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [ECR Pricing](https://aws.amazon.com/ecr/pricing/) |
| [ECR Quotas](https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html) |

## 15. 참고

- [[../ecr]]
- [[01-repository-and-image]]
- [[02-private-vs-public-vs-pull-through]]
