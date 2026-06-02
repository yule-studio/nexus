---
title: "AWS License Manager"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:44:00+09:00
tags:
  - aws
  - management
  - license
---

# AWS License Manager

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | License Manager hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

소프트웨어 라이선스 (Windows Server / SQL Server / Oracle / SAP) 추적 + BYOL 한도 강제. 멀티 account 자원에 자동 적용.

## 2. 왜 / 언제

- Windows / Oracle / SAP 라이선스 컴플라이언스
- 사용 코어 / vCPU 한도 추적 (Oracle 등)
- BYOL (Bring Your Own License) 환경

## 3. 함정

- 무료
- 라이선스 정확성은 vendor 계약 / 감사 시 확인 필요
- 작은 환경 / Linux only = 불필요

## 4. 관련

- [[../management|↑ Management]]
- [[../../security/directory-service/directory-service]]
- [[../../customer-enablement/marketplace/marketplace]]
