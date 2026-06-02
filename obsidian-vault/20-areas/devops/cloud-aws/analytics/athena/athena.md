---
title: "Amazon Athena"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:41:00+09:00
tags:
  - aws
  - analytics
  - sql
  - presto
---

# Amazon Athena

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Athena hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**S3 의 파일을 SQL 로 직접 쿼리** (Presto / Trino 기반). 서버 없이 쿼리당 과금. 데이터 레이크 ad-hoc 분석의 표준.

## 2. 왜 / 언제

- S3 에 적재된 로그 / CloudTrail / VPC Flow Log SQL 분석
- 인프라 / DB 없이 ad-hoc 분석 (서버 0)
- CUR (Cost Usage Report) raw 비용 데이터 분석
- 데이터 레이크 query layer (Glue 카탈로그 + S3)
- BI 도구 (QuickSight / Metabase / Tableau) backend

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Workgroup** | 쿼리 환경 (비용 / 권한 / 결과 위치 격리) |
| **Data Catalog** | 메타데이터 (= Glue Data Catalog) |
| **Table / Partition** | S3 경로 + 스키마 매핑, partition 으로 스캔 줄임 |
| **Format** | CSV / JSON / Parquet / ORC / Avro |
| **CTAS** | `CREATE TABLE AS SELECT` — 결과를 Parquet 으로 저장 |
| **Federated Query** | S3 외 (RDS / DynamoDB / DocumentDB) 도 쿼리 |
| **Athena for Spark** | Spark 잡도 가능 (SQL 외) |

## 4. 표준 패턴 — CloudTrail 분석

```sql
SELECT eventName, userIdentity.userName, COUNT(*) AS n
FROM cloudtrail_logs
WHERE eventTime BETWEEN '2026-06-01' AND '2026-06-02'
  AND awsRegion = 'ap-northeast-2'
GROUP BY 1, 2
ORDER BY n DESC
LIMIT 20;
```

→ S3 의 CloudTrail JSON 을 Glue 카탈로그로 등록 → Athena 에서 SQL.

## 5. 비용 / 함정

- **스캔한 데이터 TB 당 $5** — 비용은 곧 스캔 크기
- 비용 절감:
  - **Parquet / ORC** (column store) 로 저장 → 보통 10-100x 절감
  - **partition** (date / region) 으로 스캔 범위 축소
  - **partition projection** — 메타 동기화 없이 partition 인식
  - **CTAS** 로 raw → 컬럼 형식 변환
- timeout 30분
- `SELECT *` 회피 — column store 의 핵심 이점 무용
- workgroup 별 비용 제한 / 알람 권장

## 6. 관련

- [[../analytics|↑ Analytics]]
- [[../glue/glue]] — 카탈로그 + ETL
- [[../../storage/s3/s3]]
- [[../../cost/cur/cur]] — Athena 로 비용 분석
