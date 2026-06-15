---
title: "Amazon WorkMail"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:12:00+09:00
tags:
  - aws
  - business-apps
  - email
  - workmail
---

# Amazon WorkMail

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | WorkMail hub |

**[[../business-apps|↑ Business Apps & EUC]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 사내 이메일 / 캘린더 — IMAP / SMTP / Exchange ActiveSync. Google Workspace / Microsoft 365 의 AWS 대안.

## 2. 함정

- 사용 빈도 낮음 — Google Workspace / Microsoft 365 가 사실상 표준
- AWS 안 응용 (Cognito / SES) 와 통합 친화
- user / 월 $4

## 3. 관련

- [[../business-apps|↑ Business Apps & EUC]]
- [[../../messaging/ses/ses]] — 발송 자체
