---
title: "AWS Analytics (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:00:00+09:00
tags:
  - aws
  - analytics
  - data
  - hub
---

# AWS Analytics (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> 데이터 레이크 / 스트림 / 검색 / 분석 서비스 묶음. raw 저장은 [[../storage/s3/s3|S3]], 분석은 Athena / Glue / Redshift / OpenSearch.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Athena** | [[athena]] | S3 위 SQL (Presto / Trino) |
| **Glue** | [[glue]] | ETL + 카탈로그 |
| **Kinesis Data Streams** | [[kinesis]] | 실시간 스트림 (shard) |
| **Data Firehose** | [[firehose]] | 스트림 → S3 / Redshift 적재 |
| **OpenSearch** | [[opensearch]] | Elasticsearch 호환 검색 |
| **MSK** | [[msk]] | 매니지드 Kafka |
| **Redshift** | [[../database/redshift/redshift]] | 데이터 웨어하우스 (DB hub 에서) |
| **EMR / QuickSight / Lake Formation / DataZone / Flink** | (★☆☆, 별도 예정) | Hadoop / BI / 권한 / 카탈로그 / 스트림 처리 |

---

## 2. 데이터 흐름 표준

```
원천 (응용 / 로그 / IoT)
  ↓ Kinesis / MSK (실시간) 또는 직접 S3 (배치)
  ↓
S3 데이터 레이크 (Bronze → Silver → Gold)
  ↓ Glue (ETL + 카탈로그)
  ↓
┌──────────┬─────────────┬───────────────┐
Athena     Redshift      OpenSearch
(ad-hoc)   (DW / BI)     (검색 / 로그)
  ↓
QuickSight / Grafana / Metabase
```

---

## 3. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| ad-hoc SQL on S3 | Athena |
| 정기 ETL / 카탈로그 | Glue |
| 실시간 스트림 (shard, 표준) | Kinesis Data Streams |
| 스트림 → S3 적재 (코드 X) | Data Firehose |
| Kafka 호환 필요 | MSK |
| 로그 / 전문 검색 | OpenSearch |
| 대규모 분석 DW | Redshift |
| BI 대시보드 | QuickSight (또는 Metabase / Grafana) |

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../storage/s3/s3]] — 데이터 레이크 base
- [[../../data/data|↗ data 영역]]
