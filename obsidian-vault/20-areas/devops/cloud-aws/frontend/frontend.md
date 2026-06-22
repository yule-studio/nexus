---
title: "AWS Frontend Web & Mobile (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:01:00+09:00
tags:
  - aws
  - frontend
  - mobile
  - hub
---

# AWS Frontend Web & Mobile (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> SPA / SSR / 모바일 응용을 빠르게 띄우는 매니지드 서비스. 인증은 Cognito, 호스팅은 Amplify (또는 S3+CloudFront), GraphQL 은 AppSync.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Amplify** | [[amplify]] | 풀스택 (호스팅 + 인증 + API + 스토리지) |
| **Cognito** | [[../security/cognito/cognito]] | 사용자 인증 (User Pool + Identity Pool) — Security 카테고리 |
| **AppSync** | (★☆☆, 별도 예정) | 매니지드 GraphQL API |
| **Pinpoint** | [[pinpoint/pinpoint\|pinpoint]] | 모바일 푸시 / 이메일 / SMS 마케팅 캠페인 (세그먼트 / 저니) — 발송 자체는 SES |
| **Device Farm** | [[device-farm/device-farm\|device-farm]] | 실제 iOS / Android 디바이스 테스트 팜 (Appium / Espresso CI) |
| **Location Service** | [[location-service/location-service\|location-service]] | 지도 / 지오코딩 / 경로 / 지오펜스 — Google Maps / Mapbox 의 AWS 대안 |

---

## 2. 표준 구성

| 시나리오 | 추천 |
| --- | --- |
| SPA (React / Vue) 빠르게 | Amplify Hosting + Cognito |
| 정적 / SSR 직접 제어 | S3 + CloudFront + ACM + Cognito |
| GraphQL + subscription | AppSync + Cognito |
| 모바일 app 백엔드 | Amplify + Cognito + AppSync + Pinpoint |

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../storage/s3/s3]] — 정적 호스팅
- [[../network/cloudfront/cloudfront]] — CDN
- [[../security/cognito/cognito]] — 인증
- [[../../frontend/frontend|↗ frontend 영역]]
