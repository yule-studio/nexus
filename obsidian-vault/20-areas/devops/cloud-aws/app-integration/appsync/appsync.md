---
title: "AWS AppSync"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:36:00+09:00
tags:
  - aws
  - integration
  - graphql
  - appsync
---

# AWS AppSync

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | AppSync hub |

**[[../app-integration|↑ Application Integration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **GraphQL** 서버 + **WebSocket subscription**. DynamoDB / RDS / Lambda / OpenSearch / HTTP 데이터 소스 통합.

## 2. 왜 / 언제

- 모바일 / SPA 의 GraphQL 백엔드
- 실시간 협업 (subscription)
- Amplify 풀스택 응용의 API
- 여러 소스를 한 GraphQL 스키마로

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Resolver** | VTL 또는 JS — request / response mapping |
| **Data Source** | DynamoDB / Lambda / RDS / HTTP / OpenSearch |
| **Subscription** | WebSocket — DynamoDB Streams / Mutation 트리거 |
| **Auth** | Cognito / IAM / API Key / OIDC / Lambda |
| **함정** | VTL resolver 디버그 어려움 — JS resolver (2023+) 권장 |

## 4. 관련

- [[../app-integration|↑ Application Integration]]
- [[../../frontend/amplify/amplify]]
- [[../../security/cognito/cognito]]
- [[../../database/dynamodb/dynamodb]]
