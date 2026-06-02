---
title: "AWS Cost Explorer"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:53:00+09:00
tags:
  - aws
  - cost
  - analytics
---

# AWS Cost Explorer

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Cost Explorer hub |

**[[../cost|↑ Cost Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

비용 / 사용량 **시계열 분석 + 예측**. 어떤 서비스가 / 어떤 태그가 / 어떤 account 가 / 시간이 갈수록 얼마 쓰는지 본다.

---

## 2. 왜 / 언제

- 이번 달 비용 폭증 원인 찾기
- 서비스별 / 태그별 / account 별 비용 분포
- 향후 3개월 비용 예측
- Savings Plans / RI 추천 검토 (Recommendations)
- 비용 이상치 탐지 (Cost Anomaly Detection)
- 정기 비용 리포트 자동 생성

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Dimension** | 분석 축 — Service / Region / Account / Tag / Usage Type / Instance Type / Charge Type |
| **Granularity** | Hourly / Daily / Monthly |
| **Filter** | 특정 dimension 값으로 좁히기 |
| **Group by** | dimension 으로 stack chart |
| **Forecast** | 향후 12개월 예측 |
| **Anomaly Detection** | ML 기반 이상치 알람 |
| **Recommendations** | Savings Plans / RI / rightsizing 추천 |
| **Reports** | 저장된 view (URL 공유 가능) |

---

## 4. 표준 분석 5 가지

| 질문 | 분석 |
| --- | --- |
| **"이번 달 어디에 가장 많이 썼나"** | Group by Service, This month |
| **"어느 환경이 비싼가"** | Group by Tag (`Env`), monthly |
| **"NAT Gateway / Data Transfer 가 얼마"** | Filter by Usage Type Group |
| **"어느 EC2 instance type 이 비싼가"** | Filter Service=EC2, Group by Instance Type |
| **"어느 account 가 가장 많이 쓰나"** | Group by Linked Account (Organizations) |

---

## 5. Cost Anomaly Detection

- ML 기반 — 일반 패턴에서 벗어난 비용 spike 자동 탐지
- monitor 종류:
  - **AWS services** — 모든 서비스
  - **Linked account** — account 별
  - **Cost category** — 사용자 정의 그룹
  - **Cost allocation tag** — 태그별 (예: 프로젝트)
- SNS 로 알림 → 이메일 / Slack
- 무료

---

## 6. 태그 비용 분석 — 핵심

```
표준 태그
  Project = <repo-name>
  Owner   = <team>
  Env     = prod / staging / dev
  Cost-Center = <부서>
```

→ 모든 자원에 일관 태그. 그리고 **Cost Allocation Tags** 활성화 (24시간 후 반영). 이후 Cost Explorer 에서 group by 가능.

→ 자세히 → [[../cost]]

---

## 7. 비용 / 함정

- **활성화 필요** — 콘솔에서 한 번 활성화 (24시간 데이터 처리).
- **무료** (UI). 단 API 호출은 $0.01 / API request (자동화 시 주의).
- **태그 미적용 자원** — `untagged` 로 보여서 분류 어려움. tag policy + Config 룰로 강제.
- **Granularity = Hourly** 는 14일치만. 더 자세히는 **CUR (Cost and Usage Report)** → Athena.
- **prod 데이터 1주 지연** — 실시간 아님. 어제 비용은 오늘 보임.

---

## 8. 관련

- [[../cost|↑ Cost Management]]
- [[../budgets/budgets]]
- [[../billing/billing]]
- [[../savings-plans/savings-plans]]
