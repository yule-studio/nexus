---
title: "Amazon OpenSearch Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:45:00+09:00
tags:
  - aws
  - analytics
  - search
  - opensearch
---

# Amazon OpenSearch Service

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | OpenSearch hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

> Elasticsearch 7.10 의 fork (라이선스 분쟁 후). AWS 가 OSS 진영을 이끔.

---

## 1. 한 줄

매니지드 **OpenSearch / Elasticsearch + Kibana 호환 Dashboards**. 로그 검색, 전문 검색 엔진, 벡터 검색 (kNN). LLM RAG 의 vector store 로도 자주 쓰임.

## 2. 왜 / 언제

- 로그 / 이벤트 검색 (CloudWatch Logs 대안 / 보조)
- 전문 검색 (응용 안 product / 문서 검색)
- 벡터 검색 (kNN, LLM RAG)
- ELK 스택을 AWS 매니지드로
- BI / SIEM 백엔드

## 3. 두 가지 배포

| | Managed Cluster | Serverless |
| --- | --- | --- |
| 모델 | 노드 (data / master / UltraWarm) | 용량 자동 |
| Setup | 노드 타입 + 수 결정 | 없음 |
| 비용 | 노드 시간 + EBS | OCU 시간 (인덱싱 / 검색 분리) |
| 권한 | IAM / fine-grained / Cognito | IAM |
| 용도 | 안정 워크로드, 큰 시간 | 변동 큰 워크로드, 빠른 시작 |

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Index** | 데이터 묶음 (RDB 의 테이블 비슷) |
| **Document** | JSON (RDB 의 row) |
| **Mapping** | 스키마 정의 (analyzer / type) |
| **Shard / Replica** | 분산 + 가용성 |
| **kNN plugin** | 벡터 검색 (HNSW) |
| **Dashboards** | Kibana 호환 시각화 |
| **Index State Management (ISM)** | hot/warm/cold/delete 자동 lifecycle |

## 5. 표준 패턴 — 로그 검색

```
응용 → CloudWatch Logs → subscription filter → Firehose → OpenSearch
응용 → (직접) Firehose → OpenSearch
응용 → Fluent Bit / Fluentd → OpenSearch
```

→ ISM 으로 30일 후 UltraWarm, 90일 후 Cold, 180일 후 삭제.

## 6. 비용 / 함정

- 노드 시간당 ($0.07~/시간) + EBS 비용 — **항상 켜져 있어 비쌈**
- UltraWarm / Cold tier 로 옛 데이터 비용 절감 (3-7x)
- **샤드 너무 많으면 메모리 폭발** — 한 shard 10~50GB 권장
- **mapping explosion** (필드 너무 많음) — 정리 안 하면 cluster 죽음
- TLS / VPC / IAM / Cognito 권한 잘못 설정 흔함
- 벡터 검색 = 메모리 큰 노드 + nprobe / efSearch 튜닝

## 7. 관련

- [[../analytics|↑ Analytics]]
- [[../firehose/firehose]]
- [[../../observability/cloudwatch/cloudwatch]]
- [[../../ml/bedrock/bedrock]] — RAG vector store
