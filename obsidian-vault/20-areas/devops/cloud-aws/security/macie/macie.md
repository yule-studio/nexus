---
title: "Amazon Macie"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:27:00+09:00
tags:
  - aws
  - security
  - pii
  - macie
---

# Amazon Macie

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Macie hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

S3 안의 **민감 데이터 (PII / 신용카드 / 주민번호) 자동 검출** + 분류. 개인정보 컴플라이언스 (GDPR / HIPAA / PCI) 의 첫 자동화.

## 2. 왜 / 언제

- 응용이 잘못해서 S3 에 PII 흘리는지 감시
- public 버킷 / 암호화 안 된 버킷 자동 탐지
- 정기 컴플라이언스 보고
- DLP (Data Loss Prevention) 1차 layer

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Sensitive Data Discovery Job** | 한 번 또는 정기 스캔 |
| **Allow / Custom Data Identifier** | 자체 패턴 (사내 주민번호 / 사번 등) |
| **Finding** | EventBridge → SNS / Slack / 자동 대응 |
| **함정** | 스캔 GB 당 과금 — 큰 버킷은 비싸짐. 샘플링 / scope 좁히기 |

## 4. 관련

- [[../security|↑ Security]]
- [[../../storage/s3/s3]]
- [[../security-hub/security-hub]]
