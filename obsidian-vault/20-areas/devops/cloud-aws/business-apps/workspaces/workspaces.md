---
title: "Amazon WorkSpaces"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:10:00+09:00
tags:
  - aws
  - business-apps
  - daas
  - workspaces
---

# Amazon WorkSpaces

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | WorkSpaces hub |

**[[../business-apps|↑ Business Apps & EUC]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **가상 데스크톱 (DaaS)** — Windows / Amazon Linux / Ubuntu. 원격 근무자 / 외주 / BYOD 표준 환경.

## 2. 왜 / 언제

- 직원 / 외주 표준 데스크톱 일괄 배포
- BYOD — 회사 자원이 개인 PC 안 들어옴
- 글로벌 팀 (region 가까운 곳에서)
- 보안 격리 환경 필요한 직무 (금융 / 의료)

## 3. 함정

- user / 월 비용 ($25~75) — 큰 조직만 합리적
- 매번 connect 다운로드 / 인증 — UX 마찰
- Directory Service / AD 통합 필요

## 4. 관련

- [[../business-apps|↑ Business Apps & EUC]]
- [[../appstream/appstream]]
- [[../../security/directory-service/directory-service]]
- [[../../iam/identity-center/identity-center]]
