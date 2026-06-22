---
title: "AWS Cost Management (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:33:00+09:00
tags:
  - aws
  - cost
  - finops
  - hub
---

# AWS Cost Management (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> AWS 비용을 보고 / 알람 / 절감하는 도구. 처음 AWS 시작하면 **Budgets 부터** 만든다.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Billing** | [[billing]] | 청구 콘솔 — 청구서 / 결제 수단 / 청구 알림 |
| **Cost Explorer** | [[cost-explorer]] | 비용 시계열 분석 + 예측 |
| **Budgets** | [[budgets]] | 예산 + 알림 (cost / usage / RI) |
| **Savings Plans** | [[savings-plans]] | 시간당 약정 (1년/3년) → 30-70% 절감 |
| **CUR** | [[cur/cur\|cur]] | 비용 raw 데이터를 S3 에 시간별 적재 — Athena / QuickSight 정밀 분석 / chargeback 의 base 데이터 |
| **Compute Optimizer** | [[compute-optimizer/compute-optimizer\|compute-optimizer]] | EC2 / EBS / Lambda / RDS 사용 패턴 ML 분석 → rightsizing 추천 (over / under-provisioning 탐지) |

---

## 2. 절감 4 가지 축

| 축 | 방법 |
| --- | --- |
| **약정** | Savings Plans / Reserved Instance |
| **Spot** | EC2 Spot (입찰, 90% 할인 + 강제 종료) |
| **Rightsizing** | Compute Optimizer 추천 반영 |
| **자동 종료 / scale** | 야간 / 주말 EC2 종료, Auto Scaling |

추가:
- S3 Intelligent-Tiering (lifecycle 자동)
- EBS gp2 → gp3 마이그레이션
- 미사용 자원 (EIP / EBS / snapshot / load balancer) 정리
- VPC Endpoint 로 NAT Gateway 트래픽 절감

---

## 3. 비용 함정 — top 빈도

```
NAT Gateway       ~$32/월 + GB-out
Data Transfer Out $0.09/GB (인터넷 방향)
Cross-AZ DT       $0.01/GB
Idle Elastic IP   $0.005/시간
RDS Multi-AZ      single 의 2배
EKS control plane $73/월
CloudWatch Logs   ingest 비쌈
```

---

## 4. 처음 설정 — 30분

1. **Budgets** — 월 예산 + 80% / 100% 알림 ([[budgets]])
2. **Cost Explorer 활성화** ([[cost-explorer]])
3. **태그 표준** — `Project / Owner / Env`
4. **Billing 알람** — CloudWatch 청구 알람
5. **Free Tier 알림** ON

---

## 5. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../finops/finops|↗ FinOps]]
