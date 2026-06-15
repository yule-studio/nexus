---
title: "Amazon API Gateway"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:41:00+09:00
tags:
  - aws
  - network
  - api
  - serverless
---

# Amazon API Gateway

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | API Gateway hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

> 참고: [[../../app-integration/app-integration|Application Integration]] hub 에서도 등장.

---

## 1. 한 줄

매니지드 **HTTP / REST / WebSocket API 게이트웨이**. Lambda / ALB / EC2 앞에서 인증 / throttling / mapping / 캐싱 / 모니터링을 처리한다.

---

## 2. 왜 / 언제

- Lambda 를 HTTP 로 노출 (serverless 백엔드)
- 외부 API 의 인증 / rate limit / API key 발급
- 여러 백엔드를 한 도메인 뒤에 묶기
- WebSocket 서버를 매니지드로
- 응용 로직 없이 mapping / 인증만 필요할 때

---

## 3. 3 타입

| 타입 | 의미 | 언제 |
| --- | --- | --- |
| **HTTP API** | 가볍고 싸고 빠름 (v2) | 신규 디폴트, JWT / OIDC 기본 |
| **REST API** | 풍부한 기능 (request validation / API key / WAF / usage plan) | 외부 파트너에 API 판매 / 정교한 제어 |
| **WebSocket API** | 양방향 실시간 | 채팅, 알림 |

→ 모르면 **HTTP API**. REST API 는 기능 필요할 때만 (비쌈).

---

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Stage** | 배포 환경 (`dev` / `prod`) — URL 의 일부 |
| **Route / Resource** | path + method |
| **Integration** | 백엔드 (Lambda / HTTP / ALB / VPC Link / Step Functions) |
| **Authorizer** | Lambda authorizer / Cognito / JWT (HTTP API) / IAM |
| **Usage Plan + API Key** | 외부 사용자에게 quota / throttle (REST 만) |
| **Custom Domain** | `api.example.com` + ACM 인증서 |
| **Mapping Template** | VTL 로 요청/응답 변환 (REST 만) |

---

## 5. 선택 가이드 — vs alternatives

| 시나리오 | 추천 |
| --- | --- |
| Lambda + HTTP | **API Gateway HTTP API** |
| ECS / EC2 + HTTP | **ALB** (싸고 빠름) |
| 외부 사용자에게 API 판매 | API Gateway REST API + Usage Plan |
| WebSocket | API Gateway WebSocket 또는 AppSync |
| GraphQL | **AppSync** (API Gateway 아님) |
| 매우 높은 throughput / 비용 민감 | ALB → Lambda (직접) |

---

## 6. 비용 / 함정

- **HTTP API**: $1 / million calls (저렴)
- **REST API**: $3.5 / million calls + cache 비용
- **WebSocket**: connection 시간 + 메시지 수
- **함정** — REST API mapping template (VTL) 디버그 어려움 → HTTP API 권장.
- **함정** — API Gateway 앞에 CloudFront 두고 캐싱하면 비용 ↓.
- **timeout 29초** (Lambda 의 15분과 별개) — 긴 작업은 비동기 (SQS).
- **CORS** — preflight 응답 명시 설정.

---

## 7. 관련

- [[../network|↑ Network]]
- [[../../app-integration/app-integration|↗ Application Integration]]
- [[../cloudfront/cloudfront]]
- [[../../compute/lambda/lambda]] — 가장 자주 짝
- [[../../security/acm/acm]] — custom domain TLS
