---
title: "AWS Business Applications & End User Computing (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:04:00+09:00
tags:
  - aws
  - business-apps
  - end-user-computing
  - hub
---

# AWS Business Applications & End User Computing (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub (Business Applications + End User Computing 통합) |

**[[../cloud-aws|↑ AWS]]**

> 사내 협업 / DaaS / 콜센터 / 마케팅 메시지 묶음. 이메일 발송은 [[../messaging/ses/ses|SES]] (Messaging 카테고리).

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **WorkSpaces** | [[workspaces]] | 가상 데스크톱 (DaaS) |
| **WorkMail** | [[workmail]] | 매니지드 이메일 / 캘린더 |
| **Chime** | [[chime]] | 화상회의 / 메신저 |
| **Connect** | [[connect]] | 매니지드 콜센터 (옴니채널) |
| **AppStream 2.0** | [[appstream]] | 데스크톱 응용 스트리밍 |
| **WorkSpaces Web** | [[workspaces-web/workspaces-web\|workspaces-web]] | 보안 격리된 브라우저 세션 — 디바이스에 데이터가 안 남는 격리 접근 (Cloudflare / Citrix 격리 대안) |
| **Thin Client** | [[thin-client/thin-client\|thin-client]] | WorkSpaces 전용 하드웨어 씬 클라이언트 ($195) — 회사 지급 표준 디바이스, 부팅 시 바로 WorkSpaces 로그인 |
| **WorkDocs** | [[workdocs/workdocs\|workdocs]] | 매니지드 문서 공유 / 협업 — **서비스 종료 (2025-04-25)**, Google Drive / Microsoft 365 등으로 이전 |

---

## 2. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| 원격 근무자 표준 데스크톱 | **WorkSpaces** |
| 사내 콜센터 운영 | **Connect** |
| Windows 응용을 브라우저로 제공 | **AppStream 2.0** |
| 보안 격리된 웹 세션 | **WorkSpaces Web** |
| 사내 이메일 (G Suite / Office365 대안) | WorkMail (잘 안 씀) |

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../messaging/ses/ses]] — 트랜잭션 이메일
- [[../iam/identity-center/identity-center]] — 직원 SSO
