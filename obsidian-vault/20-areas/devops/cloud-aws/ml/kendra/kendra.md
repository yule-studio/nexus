---
title: "Amazon Kendra"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:16:00+09:00
tags:
  - aws
  - ml
  - search
  - kendra
---

# Amazon Kendra

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Kendra hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

기업용 **의미 검색** (semantic search). S3 / SharePoint / Confluence / ServiceNow / Salesforce 커넥터 + LLM 답변 추출.

## 2. 왜 / 언제

- 사내 문서 / 위키 통합 검색
- 답변형 검색 (FAQ 추출)
- 커넥터 30+ 종 (SaaS / 사내 시스템)

## 3. 함정 / 대안

- 매우 비쌈 — Enterprise edition $7,008 / 월부터 (큰 조직만)
- **신규 = OpenSearch + Bedrock embedding (RAG)** 가 훨씬 저렴
- Bedrock **Knowledge Bases** 가 Kendra 의 단순 대안

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../../analytics/opensearch/opensearch]]
- [[../bedrock/bedrock]]
