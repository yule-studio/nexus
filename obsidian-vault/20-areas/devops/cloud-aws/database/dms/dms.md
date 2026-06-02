---
title: "AWS Database Migration Service (DMS)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:51:00+09:00
tags:
  - aws
  - database
  - migration
  - dms
---

# AWS Database Migration Service (DMS)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | DMS hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

이종 / 동종 **DB 마이그레이션 + CDC (Change Data Capture)**. Oracle → Postgres, MySQL → Aurora, RDS → S3 (Parquet 적재) 등 20+ source/target.

## 2. 왜 / 언제

- 온프레 / 타 클라우드 DB → AWS RDS / Aurora 이전
- 다른 엔진 마이그레이션 (Oracle → Postgres) — Schema Conversion Tool (SCT) 와 함께
- CDC — DB 의 변경을 실시간으로 S3 / Kafka / OpenSearch 로 흘림
- 운영 중인 DB 무중단 이전 (full + ongoing)
- ETL (operational DB → DW)

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Replication Instance** | DMS 가 실행되는 EC2 (provisioned) 또는 Serverless |
| **Endpoint** | source / target 연결 정보 |
| **Task** | full load / CDC / full + CDC |
| **Table mapping** | 테이블 / 컬럼 필터 |
| **Transformation rule** | 이름 / 데이터 타입 변환 |
| **SCT (Schema Conversion Tool)** | 스키마 / 코드 변환 (Oracle → PG 등) |
| **DMS Serverless** | replication instance 관리 없이 |

## 4. 표준 시나리오

```
on-prem Oracle
   ↓ SCT (스키마 변환)
   ↓ DMS full load
RDS Postgres
   ↓ DMS CDC (이중화)
   ↓ cutover (응용 endpoint 전환)
```

→ full + CDC 로 다운타임 최소화 (수 분 이내).

## 5. Source / Target 매트릭스 (자주 쓰는 것)

| Source | Target |
| --- | --- |
| Oracle / SQL Server / MySQL / Postgres | RDS / Aurora 같은 / 다른 엔진 |
| MongoDB | DocumentDB / S3 |
| Kafka / Kinesis | RDS / Aurora |
| RDS / Aurora | **S3 (Parquet)** / Redshift / OpenSearch / Kafka / Kinesis |
| DynamoDB | S3 |

## 6. 비용 / 함정

- replication instance EC2 시간 (~$0.20/시간 보통) + GB-out
- DMS Serverless = DCU 시간 기반
- **LOB 컬럼** (BLOB / CLOB) — limited LOB / full LOB 모드 선택, 큰 LOB 는 chunked
- **DDL 변경** — task 시작 후 source 의 DDL 변화는 limited 지원. 마이그레이션 전 DDL 동결 권장
- target 의 인덱스 / FK 는 full load 후 만들기 (성능)
- CDC 가 source 의 binlog / archive log 활성화 필요

## 7. 관련

- [[../database|↑ Database]]
- [[../rds/rds]]
- [[../aurora/aurora]]
- [[../../storage/s3/s3]] — CDC target
