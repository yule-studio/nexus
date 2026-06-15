---
title: "Amazon EMR"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:00:00+09:00
tags:
  - aws
  - analytics
  - emr
  - hadoop
  - spark
---

# Amazon EMR

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | EMR hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **Hadoop / Spark / Hive / Presto / HBase / Flink** 클러스터. 빅데이터 분산 처리. 신규 클라우드 native = **EMR Serverless** 권장.

## 2. 왜 / 언제

- 거대 ETL / 분석 (TB ~ PB)
- 기존 Hadoop / Spark 응용을 AWS 로
- ML feature pipeline (Spark)
- 신규 = **Athena / Glue / EMR Serverless** 가 보통 더 적합

## 3. 형태

| | EMR on EC2 | EMR on EKS | EMR Serverless |
| --- | --- | --- | --- |
| 모델 | 클러스터 직접 | K8s 위 | 자동 용량 |
| Setup | 노드 타입 / 수 | EKS 위 잡 | 없음 |
| 비용 | 인스턴스 + EMR fee | EKS + EMR fee | 사용 vCPU/메모리-second |
| 신규 권장 | X | 멀티테넌트 K8s | **단순 잡** |

## 4. 관련

- [[../analytics|↑ Analytics]]
- [[../glue/glue]] / [[../athena/athena]]
- [[../../compute/eks/eks]]
