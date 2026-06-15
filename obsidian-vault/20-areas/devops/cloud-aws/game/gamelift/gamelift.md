---
title: "Amazon GameLift"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:40:00+09:00
tags:
  - aws
  - game
  - gamelift
---

# Amazon GameLift

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | GameLift hub |

**[[../game|↑ Game]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **게임 서버 호스팅 + 매치메이킹 (FlexMatch)**. 글로벌 fleet + Spot 으로 비용 절감. Unreal / Unity / 자체 엔진.

## 2. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Fleet** | EC2 / Anywhere / Spot 게임 서버 묶음 |
| **FlexMatch** | 매치메이킹 알고리즘 (latency / 스킬 기반) |
| **GameLift Realtime Servers** | 작은 게임용 lightweight (Lua) |
| **GameLift Anywhere** | 자체 인프라 (온프레 / 다른 클라우드) 와 혼합 |
| **함정** | Spot 인터럽션 — graceful shutdown 처리 필요. region 별 fleet 으로 latency |

## 3. 관련

- [[../game|↑ Game]]
- [[../../compute/ec2/ec2]]
- [[../../database/dynamodb/dynamodb]] — player session
