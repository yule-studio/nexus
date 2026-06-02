---
title: "Amazon Neptune"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:22:00+09:00
tags:
  - aws
  - database
  - graph
---

# Amazon Neptune

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Neptune hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **그래프 DB** — Property Graph (Gremlin / openCypher) + RDF (SPARQL) 둘 다. 추천 / 사기 탐지 / 지식 그래프.

## 2. 왜 / 언제

- 노드 + 관계 모델이 자연스러운 도메인 (소셜 / 추천 / 사기)
- 깊은 path / 이웃 탐색 (RDB JOIN 폭발)
- 지식 그래프 (RAG + 그래프)

## 3. 함정

- 그래프 DB 학습 곡선 (Gremlin / Cypher)
- 신규 응용 = **Neo4j (자체 운영) vs Neptune (매니지드)** 트레이드오프
- pgvector / OpenSearch 가 단순 추천에는 충분할 수도

## 4. 관련

- [[../database|↑ Database]]
- [[../../ml/bedrock/bedrock]] — 지식 그래프 + LLM
