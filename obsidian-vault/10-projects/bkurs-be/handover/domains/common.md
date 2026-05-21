---
title: "도메인 — common (횡단 인프라)"
kind: handover
status: stable
created_at: 2026-05-22T00:09:00+09:00
tags: [bkurs-be, handover, domain, common, infra]
related:
  - "[[../../bkurs-be]]"
  - "[[../common]]"
home_hub: "[[../../bkurs-be]]"
---

# common 도메인

> 도메인 무관 횡단 모듈 — envelope / error / exception / handler / web / messaging / config /
> health. **모든 도메인이 의존**하므로 시그니처 변경 영향 큼. 상세는 [[../common]]
> SSoT, 본 문서는 catalog 정리.

## 1. 책임 한눈에

```
common/
├── api/        ─ 응답 envelope (모든 응답 wrap)
├── error/      ─ ApiError / ErrorCodeSpec / FieldErrorItem
├── enums/      ─ 공통 ErrorCode (COM-*)
├── exception/  ─ BusinessException + 7 표준 서브타입
├── handler/    ─ GlobalExceptionHandler + ValidationHandler
├── web/        ─ RequestIdFilter (UUID + MDC)
├── messaging/  ─ MessagingPublisher (InMemory / Rabbit)
├── config/     ─ Security / Cors / Async / OpenApi / Time
└── health/     ─ HealthController (smoke)
```

## 2. 파일 catalog

| 경로 | 책임 |
| --- | --- |
| `api/ApiEnvelope.java` | record — invariant 강제 생성자 |
| `api/ApiEnvelopes.java` | `@Component` helper — ok / created / noContent / fail |
| `api/ApiMeta.java` | record (requestId / timestamp) |
| `error/ApiError.java` | record (code / status / message / messageKey / fieldErrors / requestId) |
| `error/ErrorCodeSpec.java` | interface — 도메인 enum 이 구현 |
| `error/FieldErrorItem.java` | record (field / code / message) |
| `enums/ErrorCode.java` | 공통 enum (`COM-4001` ~ `COM-5002`) |
| `exception/BusinessException.java` | abstract base + ErrorCodeSpec 동봉 |
| `exception/{NotFound,Conflict,InvalidInput,Unauthorized,Forbidden,ServiceUnavailable}Exception.java` | 7 표준 서브타입 |
| `handler/GlobalExceptionHandler.java` | `@RestControllerAdvice @Order(LOWEST_PRECEDENCE)` fallback |
| `handler/GlobalValidationExceptionHandler.java` | Validation 전용 Order 높임 |
| `web/RequestIdFilter.java` | UUID 발급 + MDC put + 응답 header |
| `web/RequestIds.java` | 상수 / 추출 helper |
| `messaging/MessagingPublisher.java` | interface |
| `messaging/InMemoryMessagingPublisher.java` | default log only |
| `messaging/RabbitMessagingPublisher.java` | `@ConditionalOnProperty bkurs.messaging.provider=rabbit` |
| `messaging/RabbitConfig.java` | exchange / queue / binding bean |
| `messaging/MessageTopic.java` | enum (PAYMENT_OVERDUE 등) |
| `config/SecurityConfig.java` | stateless + JWT + permitAll + role |
| `config/CorsConfig.java` | allowedOriginPatterns + credentials |
| `config/AsyncConfig.java` | 2 pool — externalCallExecutor / defaultAsyncExecutor + MDC TaskDecorator |
| `config/OpenApiConfig.java` | Swagger 8 group (auth/academy/course/record/plan/saenggibu/search/common) |
| `config/TimeConfig.java` | `Clock.systemDefaultZone()` bean (테스트 시 @MockBean 으로 고정) |
| `health/HealthController.java` | `/api/v1/health` smoke |

## 3. 공통 ErrorCode 12종 (`COM-`)

| code | status | 의미 |
| --- | --- | --- |
| COM-4001 | 400 | INVALID_INPUT |
| COM-4002 | 400 | REQUEST_BODY_NOT_READABLE |
| COM-4003 | 400 | VALIDATION_ERROR |
| COM-4004 | 401 | UNAUTHORIZED |
| COM-4005 | 403 | ACCESS_DENIED |
| COM-4006 | 404 | NOT_FOUND |
| COM-4007 | 405 | METHOD_NOT_ALLOWED |
| COM-4008 | 406 | NOT_ACCEPTABLE |
| COM-4009 | 409 | CONFLICT |
| COM-4010 | 415 | UNSUPPORTED_MEDIA_TYPE |
| COM-5001 | 500 | INTERNAL_SERVER_ERROR |
| COM-5002 | 503 | SERVICE_UNAVAILABLE |

## 4. envelope 응답 형태

```json
{
  "success": true | false,
  "data": { ... } | null,
  "error": null | { "code": "...", "status": 4xx, "message": "...", "messageKey": "...", "fieldErrors": [...], "requestId": "..." },
  "meta": { "requestId": "...", "timestamp": "2026-05-21T..." }
}
```

## 5. 변경 영향

| 변경 | 영향 |
| --- | --- |
| `ApiEnvelope` record 필드 추가 | 모든 도메인 controller / FE typings 갱신 |
| `ErrorCode` 새 항목 | 후방 호환 — 추가만은 안전 |
| `BusinessException` 시그니처 | 모든 도메인 advice 영향 — 신중 |
| `RequestIdFilter` 동작 | 로그 / MDC 의존하는 모든 코드 |
| `AsyncConfig` thread pool 정책 | gateway 호출 모두 영향 |

## 6. 의존받음

거의 **모든 도메인**. 특히:

- `api/` — 모든 controller
- `exception/` + `handler/` — 모든 도메인 advice 와 fallback chain
- `web/RequestIdFilter` — 모든 응답의 requestId
- `config/SecurityConfig` — 모든 endpoint 의 인증 / 권한
- `config/AsyncConfig` — 모든 비동기 호출

## cross-link

- [[../common]] — 상세 사용법 SSoT
- [[../overview]] — 7 layer 흐름 + filter chain
- [[../auth-and-security]] — SecurityConfig + CORS
- [[../external-gateways]] — AsyncConfig externalCallExecutor 사용 측
