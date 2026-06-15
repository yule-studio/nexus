---
title: "Amazon Managed Workflows for Apache Airflow (MWAA)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:37:00+09:00
tags:
  - aws
  - integration
  - airflow
  - mwaa
  - data-pipeline
---

# Amazon MWAA (Managed Airflow)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MWAA hub |

**[[../app-integration|↑ Application Integration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **Apache Airflow**. Python DAG 로 데이터 파이프라인 오케스트레이션. Step Functions 와 보완 (코드 / 외부 통합 강함).

## 2. 왜 / 언제

- 기존 Airflow DAG 있는 데이터팀
- Python 코드로 복잡한 ETL / ML 파이프라인
- Airflow 의 풍부한 provider 생태계 (Snowflake / Databricks / 사내 시스템)
- Step Functions 의 ASL 보다 코드가 편할 때

## 3. MWAA vs Step Functions vs Glue Workflow

| | MWAA (Airflow) | Step Functions | Glue Workflow |
| --- | --- | --- | --- |
| 정의 | Python DAG | ASL JSON | UI / Glue 잡 |
| 통합 | 매우 넓음 (커뮤니티) | AWS native 깊음 | Glue 전용 |
| 항상 켜져? | ✅ | X (실행 시만) | X |
| 비용 | 환경 시간당 ($1+) | 호출당 | Glue 잡당 |

→ AWS-only / 짧은 워크플로우 = Step Functions. **데이터 파이프라인 / Python / 풍부 통합** = MWAA.

## 4. 함정

- 시간당 비용 — idle 도 청구
- 환경 버전 업글 manual
- 신규 = **Apache Airflow on EKS** 또는 SaaS (Astronomer) 도 후보

## 5. 관련

- [[../app-integration|↑ Application Integration]]
- [[../step-functions/step-functions]]
- [[../../analytics/glue/glue]]
