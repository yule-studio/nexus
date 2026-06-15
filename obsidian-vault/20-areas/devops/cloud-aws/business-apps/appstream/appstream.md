---
title: "Amazon AppStream 2.0"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:15:00+09:00
tags:
  - aws
  - business-apps
  - euc
  - appstream
---

# Amazon AppStream 2.0

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | AppStream hub |

**[[../business-apps|↑ Business Apps & EUC]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**Windows 데스크톱 응용을 브라우저로 스트리밍** — 사용자는 응용 설치 X. SAP / AutoCAD / Photoshop 등 무거운 SW 를 웹에서.

## 2. 왜 / 언제

- 라이선스 / 보안 이슈로 응용 설치 못 시키는 환경
- BYOD 사용자에게 표준 응용
- 응용 외주 / 임시 사용자
- WorkSpaces (전체 데스크톱) 보다 응용 단위

## 3. 함정

- user 시간 단위 비용 (~$0.10/시간) — 대규모 사용 시 비쌈
- GPU 응용 (CAD 등) 은 GPU instance 비용 ↑
- 대안: Citrix / VMware Horizon

## 4. 관련

- [[../business-apps|↑ Business Apps & EUC]]
- [[../workspaces/workspaces]]
- [[../workspaces-web/workspaces-web]]
