---
title: "API Gateway — HTTP / REST / WebSocket"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T15:00:00+09:00
tags:
  - aws
  - api-gateway
  - fundamentals
---

# API Gateway — 세 가지 타입

**[[../api-gateway|↑ API Gateway]]** · **[[../../network|↑↑ Network]]**

API Gateway 는 같은 이름 안에 **3 개 별도 제품**이다. 신규 가입 시 가장 큰 결정.

---

## 1. 3 타입 비교

| | **HTTP API (v2)** | **REST API (v1)** | **WebSocket API** |
| --- | --- | --- | --- |
| 출시 | 2019 | 2015 | 2018 |
| 비용 | $1 / M 호출 | **$3.50 / M** | connection 시간 + 메시지 |
| latency | < 10ms | ~30ms | n/a (양방향) |
| 인증 | JWT (Cognito / OIDC), IAM, Lambda | + API Key + Resource Policy + Cognito User Pool | Lambda / IAM |
| Throttling | account 수준 | **route 별 + Usage Plan + API Key** | account 수준 |
| Request Validation | X | ✅ | X |
| Mapping Template (VTL) | X | ✅ | ✅ |
| Caching | X | ✅ | X |
| Private | X | ✅ | X |
| WAF | CloudFront 앞단으로 | **직접** | X |
| 권장 | **신규 디폴트** | 외부 파트너에 API 판매 / 정밀 제어 | 채팅 / 실시간 |

---

## 2. 선택 결정 트리

```
WebSocket 필요?
   YES → WebSocket API
   NO  → 외부 사용자에게 API 판매 + API Key / Usage Plan?
            YES → REST API
            NO  → request validation (JSON Schema) 필수?
                     YES → REST API
                     NO  → **HTTP API** (디폴트, 싸고 빠름)
```

→ 모르면 **HTTP API**. 나중에 REST 로 옮기는 건 큰 작업이라 미리 선택.

---

## 3. 공통 — 응용 요청 흐름

```
Client
  ↓ HTTPS
[Custom Domain (ACM)] → Route 53
  ↓
API Gateway (인증 / throttling / 변환)
  ↓
Integration (Lambda / HTTP / VPC Link / Step Functions / DynamoDB ...)
  ↓
응용
```

---

## 4. HTTP API 가 REST 대비 못 하는 것

처음 HTTP API 선택했다가 후회하는 흔한 케이스:

- **Mapping Template (VTL)** — 요청 / 응답 JSON 변환. HTTP API 는 1:1 passthrough 만.
- **Request Validation** — JSON Schema 로 body 검증. HTTP API 는 응용에서 처리.
- **API Key + Usage Plan** — 외부 사용자 quota / throttle. HTTP API 는 account 전체만.
- **Cache** — 응답 캐싱. HTTP API 없음 → CloudFront 앞단으로 회피.
- **Private API** — VPC 안에서만 접근. HTTP API 는 X.

→ 위 5 가지가 명확하면 REST API. 아니면 HTTP API.

---

## 5. WebSocket API 의 메시지 모델

```
$connect      → 연결 시 (인증 / 등록)
$disconnect   → 끊김 시 (정리)
$default      → 라우트 키 매칭 안 됨
<route key>   → 응용 정의 (예: "sendMessage")
```

- 응답은 `POST https://<api-id>.execute-api.<region>.amazonaws.com/<stage>/@connections/<connectionId>` 로
- connection 시간당 비용 — 유휴 연결도 누적

---

## 6. 관련

- [[../api-gateway|↑ API Gateway hub]]
- [[02-stages-and-deployments]]
- [[03-models-and-validation]]
- [[../02-integrations/01-lambda]]
