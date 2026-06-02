---
title: "AWS PrivateLink"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:54:00+09:00
tags:
  - aws
  - network
  - privatelink
  - vpc-endpoint
---

# AWS PrivateLink

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | PrivateLink hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

다른 VPC 또는 SaaS 서비스 (S3 / DynamoDB / Datadog / Snowflake 등) 에 **인터넷 / NAT 없이 사설 endpoint 로 접근**. AWS 서비스 호출도 VPC 안에서.

## 2. 왜 / 언제

- private subnet 에서 S3 / DynamoDB / KMS / SSM 호출 (NAT 비용 0)
- 다른 account / 다른 VPC 의 서비스 (예: 사내 ALB) 를 사설 노출
- SaaS (Datadog / Snowflake / MongoDB Atlas) 를 사설 망 안에서
- 데이터가 인터넷을 거치지 않도록 (보안 / 컴플라이언스)

## 3. 두 유형

| 유형 | 의미 | 비용 |
| --- | --- | --- |
| **Gateway Endpoint** | S3 / DynamoDB 만 | **무료** |
| **Interface Endpoint** | 그 외 모든 서비스 + 사내 / SaaS | 시간당 + GB |

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **VPC Endpoint** | 내가 만드는 사설 진입점 |
| **Endpoint Service** | 내가 노출하는 서비스 (NLB / GLB 앞단) |
| **Service Consumer / Provider** | 호출자 / 제공자 |
| **Endpoint Policy** | endpoint 통해 어떤 action / resource 허용할지 IAM-like |
| **DNS** | private DNS 활성화 시 기존 endpoint 이름 그대로 해결 |

## 5. 표준 패턴 — private subnet 에서 S3 호출

```
private subnet
   ↓ Gateway Endpoint (S3 전용)
S3
```

→ NAT Gateway 불필요. **GB-out $0.09 → 무료**. 가장 자주 쓰는 PrivateLink.

다른 패턴:
```
응용 VPC
   ↓ Interface Endpoint (com.amazonaws.region.ssm)
SSM
```

```
Consumer VPC               Provider VPC
   ↓ Interface Endpoint  →  NLB → ECS service
```

## 6. 비용 / 함정

- Gateway Endpoint (S3/DynamoDB) **무료** — 처음 만들 PrivateLink
- Interface Endpoint 시간당 ~$0.013 + GB 처리 ~$0.01 — AZ 마다 ENI
- AZ 마다 ENI 있어야 cross-AZ 트래픽 회피
- **Endpoint Policy** 안 걸면 IAM 권한만 적용 — 정밀 제한이 필요하면 Policy
- private DNS 활성화 — 기존 호출 (예: `s3.amazonaws.com`) 그대로 동작

## 7. 관련

- [[../network|↑ Network]]
- [[../vpc/vpc]]
- [[../transit-gateway/transit-gateway]]
- [[../../storage/s3/s3]]
