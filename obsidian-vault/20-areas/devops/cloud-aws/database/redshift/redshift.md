---
title: "Amazon Redshift"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:50:00+09:00
tags:
  - aws
  - database
  - data-warehouse
  - redshift
---

# Amazon Redshift

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Redshift hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **데이터 웨어하우스** (컬럼 기반, MPP). 수십 TB ~ PB 분석 쿼리. Snowflake / BigQuery 의 AWS 버전.

## 2. 왜 / 언제

- 큰 분석 워크로드 (Athena 가 늦는 규모)
- 정기 BI 보고서 / 대시보드
- 여러 부서가 같은 데이터를 쿼리
- 데이터 마트 / 별 스키마 (star schema)
- ML feature store 백엔드 (Redshift ML)

## 3. 두 가지 배포

| | Provisioned (Cluster) | Serverless |
| --- | --- | --- |
| 모델 | 노드 (RA3 / DC2) | RPU (Redshift Processing Unit) 자동 |
| 비용 | 노드 시간 | 사용 RPU-second |
| Setup | 노드 수 / 타입 | 0 |
| 용도 | 안정적 큰 워크로드 | 변동 큰 / 새 시작 |

→ **신규 = Serverless** 가 디폴트. 큰 안정 워크로드 = Provisioned + Reserved.

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Distribution Key** | row 가 어느 노드로 가는지 (key / all / even / auto) |
| **Sort Key** | 컬럼별 정렬 — range 쿼리 빠름 |
| **Columnar** | 컬럼 단위 저장 + 압축 |
| **MPP** | 모든 노드가 병렬 처리 |
| **Workload Management (WLM)** | 쿼리 우선순위 / 동시성 |
| **Spectrum** | S3 외부 데이터를 SQL 쿼리 (Athena 유사) |
| **Data Sharing** | 다른 cluster 가 데이터 읽기 (복제 없이) |
| **Materialized View** | 미리 계산된 view |
| **Redshift ML** | SQL 로 SageMaker 모델 호출 |

## 5. Redshift vs Athena vs Aurora

| | Redshift | Athena | Aurora |
| --- | --- | --- | --- |
| 용도 | DW | ad-hoc on S3 | OLTP |
| 데이터 위치 | 자체 + Spectrum (S3) | S3 만 | 자체 |
| 쿼리 latency | 빠름 (warm) | 중간 | 매우 빠름 (point) |
| 비용 모델 | cluster / RPU | TB 스캔 당 | 인스턴스 / I/O |
| Concurrency | 우수 (WLM) | 보통 | 우수 |

## 6. 비용 / 함정

- Serverless: $0.36 / RPU-hour, min 8 RPU
- Provisioned: ra3.xlplus ~$1.08/시간 (관리형) — Reserved 로 60% 절감
- 작은 워크로드는 Athena 가 훨씬 쌈
- **VACUUM / ANALYZE** — sort 유지 + 통계. Auto Vacuum 가능
- 큰 SELECT * — 컬럼스토어 이점 없어짐
- **Distribution Key 잘못** = 한 노드 hotspot
- **Data Sharing** 으로 여러 환경 (prod / analytics) 분리 가능

## 7. 관련

- [[../database|↑ Database]]
- [[../../analytics/analytics]]
- [[../../analytics/athena/athena]]
- [[../../storage/s3/s3]]
