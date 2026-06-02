---
title: "Amazon WorkSpaces Web"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:16:00+09:00
tags:
  - aws
  - business-apps
  - euc
  - workspaces-web
---

# Amazon WorkSpaces Web

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | WorkSpaces Web hub |

**[[../business-apps|↑ Business Apps & EUC]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**보안 격리된 브라우저 세션** — 사용자의 디바이스에 데이터가 남지 않음. 사내 시스템 접근 시 안전 격리.

## 2. 왜 / 언제

- 외주 / 임시 사용자에게 사내 SaaS 접근만 허용
- 다운로드 / 클립보드 제한 (DLP)
- BYOD 보안 우려 해결
- Zero-trust 응용 접근 — AWS Verified Access 와 보완

## 3. 함정

- user 시간당 비용 — 활성 시간만 청구
- Citrix / VMware / Cloudflare Browser Isolation 대안 검토

## 4. 관련

- [[../business-apps|↑ Business Apps & EUC]]
- [[../workspaces/workspaces]]
- [[../appstream/appstream]]
