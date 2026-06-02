---
title: "AWS Savings Plans"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:55:00+09:00
tags:
  - aws
  - cost
  - savings
  - commitment
---

# AWS Savings Plans

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Savings Plans hub |

**[[../cost|↑ Cost Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**시간당 $X 를 1년 / 3년 약정** 하면 EC2 / Fargate / Lambda / SageMaker 비용을 **30-70% 할인**. RI (Reserved Instance) 의 진화형 — 더 유연.

---

## 2. 왜 / 언제

- 워크로드가 **꾸준히 도는** 부분 (= baseline)
- 예측 가능한 컴퓨팅 사용량 (최소 부분만이라도)
- On-Demand 만 쓰는 건 거의 항상 손해
- 자체 datacenter 운영 비용보다 싸게 만들 때

> 신규 워크로드 = 1주 ~ 1개월 baseline 측정 → baseline 의 70% 만 Savings Plan 으로 약정.

---

## 3. 종류

| 종류 | 적용 범위 | 할인 | 유연성 |
| --- | --- | --- | --- |
| **Compute Savings Plans** | EC2 + Fargate + Lambda, **region / family / size / OS / tenancy 모두 자유** | 최대 **66%** | **가장 유연 (권장)** |
| **EC2 Instance Savings Plans** | 특정 region + family (예: `m5` in ap-northeast-2) | 최대 **72%** | size / OS 만 유연 |
| **SageMaker Savings Plans** | SageMaker | 최대 **64%** | SageMaker 안에서 유연 |

→ 거의 항상 **Compute Savings Plans** 가 정답 (유연성 우위).

---

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Hourly commitment** | 시간당 $ — 약정 단위 |
| **Term** | 1년 / 3년 |
| **Payment** | All Upfront / Partial Upfront / No Upfront (할인율 차) |
| **Coverage** | 사용량 중 SP 가 덮는 비율 (목표 80~90%) |
| **Utilization** | 약정한 SP 중 실제 사용된 비율 (목표 ~100%) |
| **Recommendations** | Cost Explorer 가 추천 (지난 7/30/60일 기반) |

---

## 5. RI vs Savings Plans 비교

| | Savings Plans | Reserved Instance (RI) |
| --- | --- | --- |
| 형태 | 시간당 $ 약정 | 인스턴스 약정 |
| 범위 | EC2 + Fargate + Lambda | EC2 / RDS / ElastiCache / Redshift / OpenSearch / ... |
| 유연성 | **높음** (instance family 바꿔도 OK) | 낮음 (family 고정) |
| 거래 (Marketplace) | X | EC2 RI 만 가능 |
| 권장 | **컴퓨팅 = 디폴트** | RDS / ElastiCache / Redshift 같이 SP 미지원 서비스 |

→ EC2 = Savings Plans. RDS / ElastiCache / Redshift = RI.

---

## 6. 표준 전략

1. **3개월 baseline 측정** — Cost Explorer 로 시간당 최저점 확인
2. **baseline 의 70~80% 만 1년 No Upfront Compute SP**
   → 사용량이 0 까지 안 떨어지므로 항상 활용
3. 안정화되면 **3년 + All Upfront** 로 추가 약정 (더 큰 할인)
4. **분기마다 utilization / coverage 점검** ([[../budgets/budgets]] 알림)

---

## 7. 함정 / 비용

- **약정 = 환불 X** — 워크로드 사라져도 시간당 $ 는 계속 청구.
- **utilization 낮으면 손해** — 약정 < 실사용 이어야 절감. **baseline 의 70%** 가 보수적 기준.
- **Lambda 도 적용** — Compute SP 가 Lambda 까지 덮음 (Lambda Provisioned Concurrency).
- **Spot 은 적용 X** — Spot 은 이미 따로 저렴.
- **Organizations** — management account 에서 SP 사면 모든 linked account 가 혜택 (sharing ON 기본).
- **Marketplace 거래 X** — RI 와 달리 SP 는 중도 판매 불가.

---

## 8. 관련

- [[../cost|↑ Cost Management]]
- [[../cost-explorer/cost-explorer]] — Recommendations 보기
- [[../budgets/budgets]] — utilization / coverage 알림
- [[../../compute/ec2/ec2]]
- [[../../compute/lambda/lambda]]
