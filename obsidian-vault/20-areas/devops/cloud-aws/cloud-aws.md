---
title: "AWS — Cloud Hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:00:00+09:00
tags:
  - devops
  - cloud
  - aws
  - hub
---

# AWS — Cloud Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.2.0 | 2026-05-19 | engineering-agent/tech-lead | 서비스별 subfolder 구조로 재편 (compute/ec2/ec2.md 형식) + iam 을 별도 top-level 카테고리로 승격 |
| v.1.1.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리별 서비스 분리 hub |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |

**[[../devops|↑ devops]]**

> Amazon Web Services — 공공 클라우드 표준. 200+ 서비스 중 **핵심만** 카테고리로 분리.

---

## 1. 카테고리 분류

| 카테고리 | 폴더 | 서비스 |
| --- | --- | --- |
| Compute | [[compute/compute]] | EC2 / ECS / EKS / Lambda / ECR / Auto Scaling |
| Storage | [[storage/storage]] | S3 / EBS / EFS |
| Database | [[database/database]] | RDS / Aurora / DynamoDB |
| Network | [[network/network]] | VPC / ALB / NLB / CloudFront / Route 53 |
| Security | [[security/security]] | KMS / Secrets Manager |
| IAM | [[iam/iam]] | IAM / (STS / Identity Center / Organizations 예정) |
| Messaging | [[messaging/messaging]] | SQS / SNS |
| Observability | [[observability/observability]] | CloudWatch / CloudTrail |

> 각 서비스는 서브폴더로 분리 (예: `compute/ecs/ecs.md`). 향후 서비스별 deep 노트 (운영 / 보안 / 비용 / 패턴) 누적.

---

## 2. 공통 — 시작 / CLI

### 2.1 가입 / 결제
1. aws.amazon.com — 가입 (신용카드 + 전화 인증)
2. **MFA 활성** (root 계정) — IAM 의 Security credentials
3. **root 계정 거의 안 씀** — admin IAM user 생성 후 그것만 사용
4. 결제 알람 (Billing → Budgets)

### 2.2 AWS CLI

```bash
brew install awscli
aws configure
aws configure sso              # 조직
```

### 2.3 절대 X
- root key 발급
- code 에 access key
- public S3 (의도 X)
- broad `*` IAM

---

## 3. Region / AZ

- **Region** — 지역 (ap-northeast-2 = Seoul)
- **AZ** — Region 안 격리 데이터센터 (보통 3-4)
- **Edge** — CloudFront POP

| | Multi-AZ | Multi-Region |
| --- | --- | --- |
| Latency | < 2ms | 100ms+ |
| DR | AZ 장애 | Region 장애 |
| 비용 | 적음 | 큼 |

---

## 4. IAM 기본

자세히 → [[iam/iam]]

User / Role / Group / Policy. **Least Privilege**.

---

## 5. 비용 절약

- **Savings Plan / RI** — 약정 (30-70%)
- **Spot Instance** — 입찰 (90%)
- **Auto Scaling**
- **S3 Intelligent-Tiering**
- **Cost Explorer + Budgets**
- 태그 (부서별 비용 분석)

---

## 6. IaC

| 도구 | 의미 |
| --- | --- |
| CloudFormation | AWS 공식 YAML |
| CDK | TypeScript / Python |
| Terraform | multi-cloud 표준 |
| Pulumi | TypeScript / Go / Python |
| SAM | Serverless |

자세히 → [[../iac/iac]]

---

## 7. 비용 함정 (의외)

```
NAT Gateway       ~$32/월 + GB-out
Data Transfer Out $0.09/GB
Cross-AZ DT       $0.01/GB
Idle Elastic IP   $0.005/시간
RDS multi-AZ      single 의 2배
EKS control plane $73/월
```

VPC Endpoint (S3 gateway 무료) 등으로 절약.

---

## 8. 학습 자료

- **AWS 공식 docs** — docs.aws.amazon.com (1순위)
- **AWS Well-Architected Framework** — 6 pillar
- **AWS Skill Builder** — 무료 + 자격증
- **A Cloud Guru** — 강의
- **AWS re:Invent** — 매년 11월 영상

---

## 9. 관련

- [[../cloud-gcp/cloud-gcp]] / [[../cloud-azure/cloud-azure]] / [[../cloud-oci/cloud-oci]]
- [[../iac/iac]]
- [[../kubernetes/kubernetes]] — EKS
- [[../devops|↑ devops]]
