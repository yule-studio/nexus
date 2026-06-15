---
title: "AWS Snowball / Snowmobile"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:23:00+09:00
tags:
  - aws
  - migration
  - snowball
---

# AWS Snowball / Snowmobile

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Snowball hub |

**[[../migration|↑ Migration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

물리 디바이스로 **페타바이트급 데이터 운반** — 네트워크로 못 옮기는 데이터. AWS 가 디바이스 보내 → 데이터 적재 → 반송 → S3 로.

## 2. 종류

| 종류 | 용량 |
| --- | --- |
| **Snowcone** | 8TB (작고 휴대) |
| **Snowball Edge Storage** | 80TB |
| **Snowball Edge Compute** | 80TB + EC2 / Lambda 가능 |
| **Snowmobile** | 100 PB (트럭, 단종 추세) |

## 3. 함정

- 네트워크가 가능하면 **DataSync 가 보통 더 쌈**
- 운반 시간 (수 일 ~ 주) 고려
- 암호화 자동 (KMS), 데이터 무결성 검증

## 4. 관련

- [[../migration|↑ Migration]]
- [[../../storage/s3/s3]]
- [[../../storage/datasync/datasync]]
