---
title: "AWS GameKit"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:42:00+09:00
tags:
  - aws
  - game
  - sdk
  - gamekit
---

# AWS GameKit

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | GameKit hub |

**[[../game|↑ Game]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

Unreal / Unity 플러그인 — 인증 / 게임 저장 / 리더보드 / 친구 / 알림 백엔드를 IaC + SDK 로 빠르게.

## 2. 함정

- 사용 빈도 낮음 — 자체 백엔드 (Lambda + DynamoDB + Cognito) 가 보통
- Unreal / Unity 통합 진가 — 별도 엔진은 사용 X

## 3. 관련

- [[../game|↑ Game]]
- [[../gamelift/gamelift]]
- [[../../security/cognito/cognito]]
- [[../../database/dynamodb/dynamodb]]
