---
title: "AWS Amplify"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:57:00+09:00
tags:
  - aws
  - frontend
  - amplify
  - serverless
---

# AWS Amplify

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Amplify hub |

**[[../frontend|↑ Frontend]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**프론트엔드 호스팅 + 백엔드 (Auth / API / Storage / Function)** 풀스택. Vercel / Netlify 의 AWS 대응. Next.js / React / Vue / Angular / Flutter 지원.

## 2. 왜 / 언제

- SPA / SSR / SSG 빠르게 배포 (GitHub push → 자동 빌드 / 배포)
- 초기 / 프로토타입 빠른 풀스택 (인증 + DB + storage 다 매니지드)
- 모바일 (React Native / Flutter) 백엔드
- Pull Request preview 환경 자동 생성
- CloudFront 전 세계 배포 + 도메인 + ACM 자동

## 3. 두 부분

| | Amplify Hosting | Amplify Backend (Gen 2) |
| --- | --- | --- |
| 의미 | 정적 / SSR 호스팅 | CDK 기반 백엔드 (Cognito + AppSync + DynamoDB + Lambda + S3) |
| 구성 | console + Git 연결 | TypeScript 코드 (`amplify/`) |
| 배포 | git push | git push 또는 CLI |
| 용도 | 표준 호스팅 | 풀스택 magic |

→ Gen 2 (2024+) 는 CDK 기반 TS 정의, Gen 1 은 CLI / GraphQL Transform.

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **App** | 한 프로젝트 (호스팅 + 백엔드) |
| **Branch / Environment** | branch 마다 별도 환경 (main / staging / pr-123) |
| **Backend Resources** | Auth (Cognito) / Data (AppSync + DynamoDB) / Storage (S3) / Function (Lambda) |
| **amplify_outputs.json** | 클라이언트가 사용할 endpoint 메타 |
| **Studio** | 시각적 UI 빌더 (선택) |

## 5. Amplify Hosting vs S3+CloudFront

| | Amplify Hosting | S3 + CloudFront |
| --- | --- | --- |
| Setup | git push 한 번 | IaC 직접 |
| SSR | ✅ (Next.js 등) | Lambda@Edge 직접 |
| PR preview | ✅ 자동 | 직접 |
| 비용 | 빌드 시간 + 호스팅 GB | 객체 + 트래픽 (보통 더 쌈) |
| 제어 | 중간 | **풀** |

→ 빠르게 = Amplify. 정밀 제어 = S3 + CloudFront ([[../../network/cloudfront/cloudfront]]).

## 6. 비용 / 함정

- 빌드 분당 $0.01, 호스팅 GB 당 $0.15 (CloudFront 포함)
- SSR (Next.js) 은 Lambda + CloudFront 가격 추가
- 대규모 트래픽 — S3 + CloudFront 가 훨씬 쌈
- 백엔드는 풀스택 magic 이라 **lock-in 가능성** — 단순 API 면 일반 Lambda + API Gateway 권장

## 7. 관련

- [[../frontend|↑ Frontend]]
- [[../../security/cognito/cognito]]
- [[../../network/cloudfront/cloudfront]]
- [[../../storage/s3/s3]]
