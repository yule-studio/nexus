---
title: "AWS Billing"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:52:00+09:00
tags:
  - aws
  - cost
  - billing
---

# AWS Billing

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Billing hub |

**[[../cost|↑ Cost Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

청구서 / 결제 수단 / 세금 / 청구 알림을 보는 콘솔. 비용 **분석** 은 [[../cost-explorer/cost-explorer|Cost Explorer]], **예산** 은 [[../budgets/budgets|Budgets]].

---

## 2. 왜 / 언제

- 이번 달 청구액 확인
- 결제 수단 (신용카드 / 송금) 등록
- 세금 정보 / 사업자 등록증 업로드
- 청구서 PDF 다운로드
- 청구 알람 (CloudWatch Billing Alarm) 활성화
- Free Tier 사용량 추적
- Organizations 결제 통합 (Consolidated billing)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Bills** | 월별 청구서 (정산일 = 매월 3일경) |
| **Payment methods** | 신용카드 / 송금 (송금은 USD / 사전 협의) |
| **Tax settings** | 사업자 / VAT / 세금 면제 |
| **Billing alerts** | CloudWatch Billing alarm (us-east-1 에서만 발행) |
| **Free Tier** | 일부 서비스 12개월 무료 + 항상 무료 (Lambda 1M/월 등) |
| **Consolidated Billing** | Organizations 의 management account 가 모든 member 청구 통합 |
| **Volume discount** | 통합 결제로 사용량이 합산되어 할인 구간 ↑ |
| **Credit** | 프로모션 / 마이그레이션 / Activate 크레딧 |

---

## 4. 처음 30분 — 비용 사고 방지

1. **Billing Preferences** → "Receive PDF Invoice" / "Receive Free Tier Alerts" 모두 ON
2. **CloudWatch Billing Alarm** (us-east-1) — `EstimatedCharges > $X` 알람 + SNS 이메일
3. **Budgets** ([[../budgets/budgets]]) — 월 예산 + 80% / 100% 알림
4. **IAM 권한** — billing 콘솔 접근은 root + 지정된 admin 만 (`aws-portal:*`)
5. **결제 수단 백업** — 메인 카드 만료 대비 sub 카드

---

## 5. Consolidated Billing — Organizations

```
management account
   ↓ 청구 통합
member1 ($1000)  +  member2 ($500)  +  member3 ($2000)
                                = total $3500
```

- 모든 member 의 사용량이 합쳐져 **volume discount** 적용
- Savings Plans / RI 도 account 간 공유 가능 (설정에 따라)
- 결제 카드는 management 만 (member 에서 결제 X)

---

## 6. 청구 알람 (CloudWatch) — 표준

```yaml
# CloudFormation
BillingAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: monthly-budget-exceeded
    Namespace: AWS/Billing
    MetricName: EstimatedCharges
    Dimensions: [{ Name: Currency, Value: USD }]
    Statistic: Maximum
    Period: 21600
    EvaluationPeriods: 1
    Threshold: 100      # USD
    ComparisonOperator: GreaterThanThreshold
    AlarmActions: [!Ref AlertTopic]
```

→ **반드시 us-east-1 region** 에 생성. 다른 region 에서는 metric 안 보임.

---

## 7. 함정 / 비용

- **무료** (Billing 콘솔 자체).
- **Billing Alarm region** — us-east-1 만. 자주 빠뜨리는 함정.
- **Tax invoice** — 사업자 등록증 + VAT 번호 입력해야 세금계산서 발행.
- **카드 거절** — 청구 실패 시 계정 정지. AWS 가 메일 보내지만 받지 못하면 자원이 정지될 수 있음.
- **member account 의 카드** — Organizations 안에서는 사용 안 됨 (management 만).
- **Free Tier 자동 해제 X** — 12개월 끝나면 자동 청구. 알림만 옴.

---

## 8. 관련

- [[../cost|↑ Cost Management]]
- [[../cost-explorer/cost-explorer]]
- [[../budgets/budgets]]
- [[../../management/organizations/organizations]]
