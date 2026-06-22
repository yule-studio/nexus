---
title: "AWS Database (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:07:00+09:00
tags:
  - aws
  - database
  - hub
---

# AWS Database (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

### 1.1 RDB

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **RDS** | [[rds]] | 매니지드 RDB (Postgres / MySQL / MariaDB / Oracle / SQL Server) |
| **Aurora** | [[aurora]] | AWS 자체 분산 RDB — Postgres / MySQL 호환 |

### 1.2 NoSQL / 인메모리

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **DynamoDB** | [[dynamodb]] | 매니지드 KV / Document NoSQL |
| **DocumentDB** | [[documentdb/documentdb\|documentdb]] | MongoDB 호환 매니지드 document DB — 기존 MongoDB 응용을 AWS 분산 스토리지 위로 |
| **Keyspaces** | [[keyspaces/keyspaces\|keyspaces]] | Cassandra 호환 (CQL) 매니지드 NoSQL — 기존 Cassandra 응용 이전 (신규는 DynamoDB 권장) |
| **ElastiCache** | [[elasticache/elasticache\|elasticache]] | 매니지드 Redis / Memcached — 인메모리 캐시 / 세션 / 분산 락 |
| **MemoryDB** | [[memorydb/memorydb\|memorydb]] | Redis 호환 + 멀티 AZ 영속 — 캐시가 아닌 primary DB 로 (인메모리 속도 + 영속성) |
| **Neptune** | [[neptune/neptune\|neptune]] | 매니지드 그래프 DB (Gremlin / openCypher / SPARQL) — 추천 / 사기 탐지 / 지식 그래프 |

### 1.3 검색·분석·특수

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Redshift** | [[redshift/redshift\|redshift]] | 매니지드 데이터 웨어하우스 (컬럼 / MPP) — Snowflake / BigQuery 의 AWS 버전 |
| **Timestream** | [[timestream/timestream\|timestream]] | 매니지드 시계열 DB — IoT 텔레메트리 / 메트릭 (InfluxDB / TimescaleDB 대안) |
| **QLDB** | [[qldb/qldb\|qldb]] | 변경 불가 원장 (immutable ledger) DB — **종료 (2025-07-31)**, Aurora system-versioned 테이블로 대체 |

### 1.4 마이그레이션

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **DMS** | [[dms/dms\|dms]] | 이종 / 동종 DB 마이그레이션 + CDC — Oracle → Postgres, 온프레 → RDS / Aurora 등 |

---

## 2. 선택

| 시나리오 | 추천 |
| --- | --- |
| 일반 OLTP / 이식성 | RDS Postgres / MySQL |
| 거대 워크로드 / 분산 storage | Aurora |
| Key-Value / 무한 확장 / AWS native | DynamoDB |
| 캐시 / 세션 / 분산 락 | ElastiCache (Redis) |
| 분석 / DW | Redshift (별도) |
| 시계열 | Timestream (별도) |
| 그래프 | Neptune (별도) |

---

## 3. DB 자체 운영 노하우는

[[../../../database/database|↗ 20-areas/database]] — Postgres / MySQL / Redis 자체.
여기는 **AWS 측면** (관리형 옵션 / 결제 / multi-AZ / backup 등).

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../../database/database]]
