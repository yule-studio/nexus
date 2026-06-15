---
title: "AWS Game Tech (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:03:00+09:00
tags:
  - aws
  - game
  - hub
---

# AWS Game Tech (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **GameLift** | [[gamelift]] | 매니지드 게임 서버 + 매치메이킹 |
| **GameKit** | (hub 안) | Unreal / Unity 백엔드 SDK |
| **Open 3D Engine (O3DE)** | (hub 안) | 오픈소스 게임 엔진 (Linux Foundation) |

---

## 2. 표준 게임 서버 구성

```
플레이어 클라이언트
   ↓ FlexMatch (매칭)
GameLift Fleet (게임 세션 서버)
   ↓
DynamoDB (플레이어 데이터)
S3 (replay / asset)
```

→ Spot fleet 으로 70% 절감 가능. 글로벌 fleet 으로 region-aware 매칭.

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../compute/ec2/ec2]]
- [[../database/dynamodb/dynamodb]]
