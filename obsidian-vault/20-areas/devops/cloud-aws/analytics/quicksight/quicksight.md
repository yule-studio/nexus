---
title: "Amazon QuickSight"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:01:00+09:00
tags:
  - aws
  - analytics
  - bi
  - quicksight
---

# Amazon QuickSight

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | QuickSight hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **BI 대시보드**. Tableau / Power BI / Metabase 의 AWS 대안. user 당 월 단위 + SPICE 인메모리 엔진.

## 2. 왜 / 언제

- 사내 대시보드 (RDS / Redshift / Athena / S3 소스)
- 응용에 대시보드 임베드 (Embedded Analytics)
- Q (자연어 질의 → 차트)
- 비용 분석 (CUR + Athena + QuickSight 표준)

## 3. 함정

- Reader $0.30 / 세션 (max $5/월/user), Author $24/월
- SPICE 메모리 한계 — 큰 데이터는 direct query 권장
- 차트 유연성 Tableau / Power BI 보다 약함
- Metabase / Grafana 가 더 가볍고 무료 옵션

## 4. 관련

- [[../analytics|↑ Analytics]]
- [[../athena/athena]] / [[../../database/redshift/redshift]]
- [[../../cost/cur/cur]]
