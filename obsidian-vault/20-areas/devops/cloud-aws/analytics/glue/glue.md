---
title: "AWS Glue"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:42:00+09:00
tags:
  - aws
  - analytics
  - etl
  - catalog
---

# AWS Glue

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Glue hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **ETL (Spark / Python shell) + 데이터 카탈로그**. Athena / Redshift / EMR 의 공통 메타데이터 store + 스키마 자동 발견 (Crawler).

## 2. 왜 / 언제

- S3 raw 데이터 → 정제 (Parquet 변환 / partition) → Athena / Redshift
- 여러 소스 (RDS / S3 / DynamoDB) 의 카탈로그 통합
- 정기 ETL (매일 자정 새 데이터)
- 스키마 자동 발견 (Crawler 가 S3 폴더 보고 테이블 등록)
- Data Quality 룰 (Glue DQ)

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Data Catalog** | 모든 테이블 / 스키마 / partition 메타 (Athena / EMR 공유) |
| **Crawler** | S3 / DB 스캔 → 자동 테이블 등록 |
| **Job** | ETL 실행 (Spark / Python shell / Ray) |
| **Trigger** | 스케줄 / 이벤트 / on-demand 잡 실행 |
| **DataBrew** | 노코드 데이터 정제 (UI 기반) |
| **Studio** | Notebook + Glue ETL 비주얼 편집 |
| **Workflow** | 여러 잡 + 트리거 묶음 |
| **DPU** | Spark 워커 단위 (4 vCPU + 16GB) |

## 4. 표준 패턴 — Bronze → Silver → Gold

```
S3 Bronze (raw JSON)
     ↓ Glue Crawler (스키마 등록)
     ↓ Glue Job (Parquet 변환 + partition)
S3 Silver (정제)
     ↓ Glue Job (집계)
S3 Gold (집계 / 비즈니스 view)
     ↓
Athena / QuickSight / Redshift Spectrum
```

## 5. 비용 / 함정

- ETL 잡 DPU-hour 당 $0.44 (기본). Job bookmark 로 새 데이터만 처리.
- Crawler 도 DPU 당 비용. 큰 S3 prefix 는 schedule 조심.
- Data Catalog — 1M 객체 무료, 이후 월 $1 / 100K
- **Glue 4.0 (Spark 3.3)** 이상 권장. 옛 Glue 2.x 는 deprecated.
- 단순 ETL = Lambda / Athena CTAS 가 더 쌀 수 있음. 큰 ETL = Glue.
- partition 폭발 (수만 partition) — partition projection 또는 큰 묶음

## 6. 관련

- [[../analytics|↑ Analytics]]
- [[../athena/athena]]
- [[../../storage/s3/s3]]
- [[../../database/redshift/redshift]]
