---
title: "Amazon Data Firehose"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:44:00+09:00
tags:
  - aws
  - analytics
  - streaming
  - firehose
---

# Amazon Data Firehose

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Firehose hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

> 옛 이름: **Amazon Kinesis Data Firehose** (2024 rebrand).

---

## 1. 한 줄

스트림 → **S3 / Redshift / OpenSearch / Splunk / HTTP endpoint** 적재를 코드 없이 매니지드로. 버퍼 / 압축 / Parquet 변환 / 암호화까지 처리.

## 2. 왜 / 언제

- 응용 로그 / 메트릭을 S3 에 자동 적재 (Athena 분석 준비)
- Kinesis Stream → 다른 destination 변환 / 적재
- CloudWatch Logs → S3 long-term 보관
- 3rd party (Datadog / Splunk / New Relic) 로 스트리밍
- Lambda transform 으로 record 변환 가능

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Source** | Direct PUT / Kinesis Stream / MSK / CloudWatch Logs / WAF / EventBridge |
| **Destination** | S3 / Redshift / OpenSearch / Splunk / HTTP / Datadog / MongoDB Atlas |
| **Buffer** | size (MB) 또는 interval (sec) — 작을수록 latency ↓ |
| **Transform** | Lambda 가 record 가공 (선택) |
| **Format conversion** | JSON → Parquet / ORC (Glue 스키마 필요) |
| **Dynamic partitioning** | record 값 기반 S3 prefix (`/year/month/...`) |

## 4. 표준 패턴 — 로그 적재

```
응용 로그
   ↓ Firehose Direct PUT
   ↓ Lambda Transform (가공)
   ↓ Parquet 변환
S3 (year=2026/month=06/day=01/...)
   ↓
Athena / OpenSearch
```

## 5. Kinesis Streams vs Firehose

| | Kinesis Streams | Firehose |
| --- | --- | --- |
| 모델 | shard / consumer | source → destination |
| Replay | ✅ retention | X (적재만) |
| 응용 컨슈머 | Lambda / Flink | 직접 X |
| Setup | shard 관리 | **0 관리** |
| 용도 | 실시간 처리 | **적재 자동화** |

→ 처리 = Streams. **적재만** = Firehose.

## 6. 비용 / 함정

- $0.029 / GB ingest (S3 destination) — 매우 저렴
- 추가: Parquet 변환 / Lambda transform 별도 비용
- **buffer 60s / 5MB 가 기본** — 작게 잡으면 작은 파일 폭발
- S3 적재 후 작은 파일은 합치는 ETL (Glue) 필요할 수도
- timeout / retry 정책 — destination 응답 안 좋으면 fail → S3 backup

## 7. 관련

- [[../analytics|↑ Analytics]]
- [[../kinesis/kinesis]]
- [[../opensearch/opensearch]]
- [[../../storage/s3/s3]]
