---
title: "AWS Budgets"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:54:00+09:00
tags:
  - aws
  - cost
  - budget
  - alerts
---

# AWS Budgets

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Budgets hub |

**[[../cost|↑ Cost Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**예산 + 알림 + 자동 액션**. "월 $100 넘으면 메일", "예산 100% 도달 시 EC2 자동 종료" 같은 룰. AWS 시작 30분 안에 반드시 설정.

---

## 2. 왜 / 언제

- 비용 사고 (한밤중 잘못된 EC2, GPU 인스턴스 까먹기) 방지
- 부서 / 프로젝트별 예산 추적
- Savings Plans / RI 사용률 알림 (낭비 방지)
- 비용 이상 시 자동 액션 (IAM policy / EC2 stop)
- 매월 보고 (PDF / 이메일)

> 첫 AWS 가입 직후 = **Budgets 1개라도 만들기**. 5분 작업으로 청구 사고 막음.

---

## 3. 예산 종류

| 종류 | 의미 |
| --- | --- |
| **Cost** | $ 기반 ($100/월) |
| **Usage** | 사용량 기반 (예: EC2 hours, S3 GB) |
| **Reservation utilization** | RI 사용률 < 80% 면 알림 |
| **Reservation coverage** | RI 가 워크로드의 N% 덮는지 |
| **Savings Plans utilization / coverage** | Savings Plan 도 동일 |

---

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Threshold** | 예산 대비 % (50% / 80% / 100% / 120%) |
| **Actual vs Forecasted** | 실제 도달 vs 예측 도달 — 둘 다 알림 가능 |
| **Notification** | SNS / 이메일 (최대 10명) |
| **Budget Actions** | IAM policy attach / EC2/RDS stop 자동 실행 |
| **Filter** | service / account / tag / region 별 |

---

## 5. 표준 설정

```yaml
# CloudFormation 예시
MonthlyBudget:
  Type: AWS::Budgets::Budget
  Properties:
    Budget:
      BudgetName: monthly-100usd
      BudgetType: COST
      TimeUnit: MONTHLY
      BudgetLimit: { Amount: 100, Unit: USD }
    NotificationsWithSubscribers:
      - Notification:
          NotificationType: ACTUAL
          ComparisonOperator: GREATER_THAN
          Threshold: 80
        Subscribers: [{ SubscriptionType: EMAIL, Address: ops@y }]
      - Notification:
          NotificationType: FORECASTED
          ComparisonOperator: GREATER_THAN
          Threshold: 100
        Subscribers: [{ SubscriptionType: EMAIL, Address: ops@y }]
```

→ 80% 도달 시 + 예측이 100% 넘을 때, 두 단계 알림.

---

## 6. Budget Actions — 자동 차단

예산 100% 초과 시 자동:

| 액션 | 효과 |
| --- | --- |
| **Attach IAM policy** | 사용자 / role 에 deny 정책 강제 (`Deny ec2:RunInstances`) |
| **Attach SCP** | OU / account 에 deny |
| **Stop EC2 / RDS** | 지정된 자원 일시 중지 |

→ sandbox / 학습 account 에서 강력. prod 에는 신중 (서비스 다운 가능).

---

## 7. 처음 만들 추천 예산 3개

1. **monthly-overall** — 월 전체 비용 예산
2. **dev-account** — dev OU 만 (Org 가 있으면)
3. **anomaly** — `Forecasted > 150%` (이상 사용 조기 탐지)

---

## 8. 함정 / 비용

- **첫 2개 무료**, 그 이후 $0.02 / day / budget.
- **알림 지연** — 비용 데이터가 8~24시간 지연되어 갱신. 실시간 아님.
- **예산이 막아주지 X** — 알림만. 자원은 그대로 동작. 차단하려면 Budget Action 명시.
- **이메일 한계 10명** — 더 필요하면 SNS topic 1개 + 거기 구독.
- **tag 기반 예산** — Cost Allocation Tag 활성화 필요.

---

## 9. 관련

- [[../cost|↑ Cost Management]]
- [[../cost-explorer/cost-explorer]]
- [[../billing/billing]]
- [[../savings-plans/savings-plans]]
