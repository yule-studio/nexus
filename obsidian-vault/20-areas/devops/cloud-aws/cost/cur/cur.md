---
title: "AWS Cost and Usage Report (CUR)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:58:00+09:00
tags:
  - aws
  - cost
  - cur
  - finops
---

# AWS Cost and Usage Report (CUR)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CUR hub |

**[[../cost|↑ Cost Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**비용 raw 데이터를 S3 에 시간별로 적재**. Cost Explorer 로 안 보이는 정밀 분석 (Athena / QuickSight / 자체 BI) 의 base 데이터.

## 2. 왜 / 언제

- Cost Explorer 의 default 분석을 넘는 깊이가 필요할 때
- 사내 BI / dashboard 에 비용 데이터 통합
- 부서별 / 프로젝트별 정밀 chargeback
- 분 단위 / 자원 단위 비용 — 누가 / 어떤 EC2 가 얼마를?
- 시계열 분석 / ML 비용 예측

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **CUR v2 / Data Exports** | 2024+ 권장 — 더 풍부한 컬럼 + FOCUS 표준 |
| **Granularity** | Hourly / Daily / Monthly |
| **Delivery** | S3 bucket 으로 시간별 Parquet / CSV |
| **Resource ID** | 자원 단위 비용 (옵션, 큼) |
| **Schema** | 100+ 컬럼 (서비스 / 사용량 / 단가 / 태그 / 약정) |
| **FOCUS** | 멀티클라우드 비용 표준 — AWS / GCP / Azure 공통 컬럼 |

## 4. 표준 통합 — Athena + QuickSight

```
CUR (Parquet) → S3
   ↓ Glue Crawler (테이블 등록)
   ↓
Athena
   ↓
QuickSight 또는 Metabase / Tableau / Grafana
```

→ SQL 로 자유 분석:

```sql
SELECT line_item_product_code AS service,
       resource_tags['user:Project'] AS project,
       SUM(line_item_unblended_cost) AS cost
FROM cur
WHERE billing_period = '2026-06'
GROUP BY 1, 2
ORDER BY cost DESC;
```

## 5. 비용 / 함정

- CUR 자체는 **무료** — S3 저장 / Athena 쿼리 비용만
- resource ID 포함 = 파일 매우 큼 (수십 GB / 월) — 필요한 컬럼만 / partition 활용
- 데이터 도착 6-24h 지연
- 비용 컬럼: `unblended` (개별 account 기준) vs `blended` (Org 평균) vs `amortized` (RI / SP 평균) — 보통 `amortized` 권장
- Cost Explorer 의 view 와 CUR 합계가 약간 다를 수 있음 (반올림 / 시점)

## 6. 관련

- [[../cost|↑ Cost Management]]
- [[../cost-explorer/cost-explorer]]
- [[../compute-optimizer/compute-optimizer]]
- [[../../analytics/athena/athena]]
