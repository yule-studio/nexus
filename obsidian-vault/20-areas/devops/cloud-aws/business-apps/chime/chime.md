---
title: "Amazon Chime"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:13:00+09:00
tags:
  - aws
  - business-apps
  - chime
  - voip
---

# Amazon Chime

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Chime hub |

**[[../business-apps|↑ Business Apps & EUC]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

화상회의 / 메신저 + **Chime SDK** (응용에 화상 / 음성 / SMS 임베드). Zoom / Webex 대안.

## 2. 두 갈래

| | Chime | Chime SDK |
| --- | --- | --- |
| 형태 | 완성 응용 | API / SDK (응용 안 빌트인) |
| 용도 | 사내 회의 | 텔레헬스 / 콜센터 / 라이브 강의 응용 |

→ 신규 사용 = **Chime SDK** (응용 통합). 사내 미팅 = Zoom / Google Meet 가 보통.

## 3. 함정

- 회의 분 단위 + SDK call 단위 비용
- 신규 미팅 도구로 채택 빈도 낮음 — SDK 가 핵심

## 4. 관련

- [[../business-apps|↑ Business Apps & EUC]]
- [[../connect/connect]]
