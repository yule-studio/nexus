---
title: "AWS Artifact"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:42:00+09:00
tags:
  - aws
  - security
  - compliance
  - artifact
---

# AWS Artifact

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Artifact hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 의 **컴플라이언스 보고서** (SOC 1/2/3, ISO 27001, PCI DSS, HIPAA, FedRAMP, K-ISMS 등) 다운로드 + AWS 와 NDA / BAA 계약.

## 2. 왜 / 언제

- 감사 / 계약 시 AWS 의 인증서 / 보고서 제출
- HIPAA BAA, K-ISMS 한국 인증 확인
- vendor risk assessment (회사 보안 평가)

## 3. 함정

- 무료, console 에서 직접 다운로드
- 모든 region / 모든 서비스가 같은 인증 X — 서비스별 / region 별 인증 범위 확인
- 일부 보고서는 NDA 수락 후 다운로드

## 4. 관련

- [[../security|↑ Security]]
- [[../security-hub/security-hub]]
- [[../../management/config/config]]
