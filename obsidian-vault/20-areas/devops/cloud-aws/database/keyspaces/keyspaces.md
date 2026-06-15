---
title: "Amazon Keyspaces (for Apache Cassandra)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:50:00+09:00
tags:
  - aws
  - database
  - cassandra
  - keyspaces
---

# Amazon Keyspaces

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Keyspaces hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**Cassandra 호환** (CQL) 매니지드 NoSQL. 기존 Cassandra 응용을 AWS 매니지드로.

## 2. 함정 / 대안

- Cassandra API 호환 — 100% 일치는 X (일부 함수 차이)
- 신규 응용 = **DynamoDB** (AWS native, 더 저렴)
- 기존 Cassandra 운영 부담 회피 시에만

## 3. 관련

- [[../database|↑ Database]]
- [[../dynamodb/dynamodb]]
