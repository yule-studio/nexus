---
title: "bkurs-be — common 모듈 레퍼런스"
kind: handover
status: stable
created_at: 2026-05-21T22:15:00+09:00
tags: [bkurs-be, handover, common, envelope, error-handling]
related:
  - "[[../bkurs-be]]"
  - "[[overview]]"
  - "[[auth-and-security]]"
home_hub: bkurs-be
---

# bkurs-be — common 모듈 레퍼런스

> `com.bkurs.common.*` — 도메인 무관 횡단 모듈. **모든 도메인이 의존하므로 시그니처 변경 시
> 영향 큼.** 신규 도메인 추가 시 "어디서 envelope/예외/요청ID 가 들어오는가" 를 본 문서로 확인.

## 0. 패키지 한눈에

```
common/
├── api/        # 응답 envelope (ApiEnvelope / ApiEnvelopes / ApiMeta)
├── error/      # ApiError / ErrorCodeSpec / FieldErrorItem
├── enums/      # 공통 ErrorCode (COM-*)
├── exception/  # BusinessException + 7 표준 서브타입
├── handler/    # GlobalExceptionHandler / GlobalValidationExceptionHandler
├── web/        # RequestIdFilter / RequestIds
├── messaging/  # MessagingPublisher / RabbitConfig / MessageTopic
├── config/     # SecurityConfig / CorsConfig / AsyncConfig / OpenApiConfig / TimeConfig
└── health/     # HealthController
```

## 1. `api/` — 응답 envelope

### 1.1 `ApiEnvelope<T>` (record)

모든 REST 응답을 감싼다. **invariant** (record 생성자에서 강제):

- `success=true` ⇒ `data != null ∧ error = null`
- `success=false` ⇒ `data = null ∧ error != null`
- `meta != null` 항상

```java
public record ApiEnvelope<T>(boolean success, T data, ApiError error, ApiMeta meta) { ... }
```

### 1.2 `ApiMeta` (record)

- `requestId` (String) — `RequestIdFilter` 가 부여한 UUID. 클라이언트 / 로그 추적용.
- `timestamp` (Instant) — `Clock` 기반 (테스트 시 고정 가능).

### 1.3 `ApiEnvelopes` (@Component, helper)

controller / handler 가 envelope 생성 시 직접 호출.

```java
@RequiredArgsConstructor
@RestController
public class HomeworkOwnerController {
    private final ApiEnvelopes envelopes;
    private final HomeworkService service;

    @PostMapping
    public ResponseEntity<ApiEnvelope<HomeworkResponse>> create(
        @Valid @RequestBody CreateHomeworkRequest req,
        HttpServletRequest http) {
        HomeworkResponse view = service.create(req);
        return envelopes.created(URI.create("/v1/.../homework/" + view.id()), view, http);
    }
}
```

helper 메서드 4개:

| 메서드 | HTTP | 사용 |
| --- | --- | --- |
| `ok(data, request)` | 200 | 일반 조회 / 업데이트 |
| `created(uri, data, request)` | 201 + Location | 생성 |
| `noContent()` | 204 | DELETE / 빈 응답 |
| `fail(status, error, request)` | 4xx / 5xx | handler 가 사용 — controller 직접 호출 금지 |

> **`Clock` 주입**: `TimeConfig` 가 `Clock.systemDefaultZone()` bean 등록. 테스트는 `@MockBean Clock` 으로 고정.

## 2. `error/` + `enums/` — 에러 코드

### 2.1 `ErrorCodeSpec` (interface)

도메인별 enum 이 구현. 4 필드:

- `status()` — HttpStatus
- `code()` — `{도메인}-{HTTP}{seq}` (예: `ACA-4040`)
- `messageKey()` — i18n key (예: `error.academy.not_found`)
- `defaultMessage()` — 사람 메시지 fallback

### 2.2 `ErrorCode` (공통 enum, prefix `COM-`)

| code | status | 의미 |
| --- | --- | --- |
| COM-4001 | 400 | INVALID_INPUT |
| COM-4002 | 400 | REQUEST_BODY_NOT_READABLE (JSON 깨짐) |
| COM-4003 | 400 | VALIDATION_ERROR (`@Valid` 실패) |
| COM-4004 | 401 | UNAUTHORIZED |
| COM-4005 | 403 | ACCESS_DENIED (`@PreAuthorize` 거부) |
| COM-4006 | 404 | NOT_FOUND |
| COM-4007 | 405 | METHOD_NOT_ALLOWED |
| COM-4008 | 406 | NOT_ACCEPTABLE |
| COM-4009 | 409 | CONFLICT |
| COM-4010 | 415 | UNSUPPORTED_MEDIA_TYPE |
| COM-5001 | 500 | INTERNAL_SERVER_ERROR |
| COM-5002 | 503 | SERVICE_UNAVAILABLE |

### 2.3 도메인 ErrorCode 명명

prefix 충돌 금지. 현재 점유:

| prefix | 도메인 |
| --- | --- |
| `COM-` | common |
| `AUT-` | auth |
| `ACA-` | academy |
| `ANN-` | announcement |
| `STU-` | student |
| `PLA-` / `PAY-` | plan / payment |
| `CRS-` | course |
| `HMW-` | homework |
| `STA-` | statement |
| `QUE-` | question |
| `ITV-` | interview |
| `REC-` | record |
| `DIS-` | discussion |
| `RDN-` | reading |
| `PEV-` | performance |
| `SIM-` | simulation |
| `SCH-` | search |
| `STS-` | stats |
| `DSH-` | dashboard |
| `CON-` | concept |
| `SHN-` | suhaeng |
| `SGB-` | saenggibu |
| `ADM-` | admission |
| `FIL-` `PDF-` `NOT-` `AI-` `STT-` | gateway |

> 신규 도메인 추가 시 본 표 갱신.

### 2.4 `ApiError` + `FieldErrorItem`

```java
public record ApiError(
    String code,
    int status,
    String message,
    String messageKey,
    List<FieldErrorItem> fieldErrors,
    String requestId
) { ... }
```

`fieldErrors` 는 Bean Validation 결과 (`@NotBlank`, `@Max`, ...). 생성: `ApiError.of(spec, message, fieldErrors, requestId)`.

## 3. `exception/` — 비즈니스 예외

`BusinessException` (abstract) — 모든 도메인 예외의 base. 7 표준 서브타입:

| 서브타입 | 기본 ErrorCode | 매핑 |
| --- | --- | --- |
| `NotFoundException` | NOT_FOUND | 404 |
| `ConflictException` | CONFLICT | 409 |
| `InvalidInputException` | INVALID_INPUT | 400 |
| `UnauthorizedException` | UNAUTHORIZED | 401 |
| `ForbiddenException` | ACCESS_DENIED | 403 |
| `ServiceUnavailableException` | SERVICE_UNAVAILABLE | 503 |
| `BusinessException` (직접) | 도메인 정의 | spec.status() |

도메인 예외는 보통 `extends BusinessException` 으로 ErrorCodeSpec 주입:

```java
public class AcademyNotFoundException extends BusinessException {
    public AcademyNotFoundException(long id) {
        super(AcademyErrorCode.ACADEMY_NOT_FOUND, "academy id=" + id + " not found");
    }
}
```

## 4. `handler/` — 전역 예외 처리

### 4.1 `GlobalExceptionHandler`

`@RestControllerAdvice @Order(Ordered.LOWEST_PRECEDENCE)` — 도메인 advice 가 먼저 처리, 남은 것 fallback.

처리하는 예외 (자세히 [[overview#6]]):

- `BusinessException` → spec.status() / spec.code()
- `MethodArgumentNotValidException` → 400 + fieldErrors
- `HttpMessageNotReadableException` → 400
- `AccessDeniedException` → 403
- `AuthenticationCredentialsNotFoundException` → 401
- 405 / 406 / 415 / 404

> **`Exception.class` fallback 없음** — 의도. 자세히 [[overview#6]].

### 4.2 `GlobalValidationExceptionHandler`

B4A 에서 추가. ERROR dispatch 시 `/error` 에서 401 로 떨어지는 함정 회피용. `@Order` 가 Global 보다 높음.

### 4.3 도메인 advice 패턴

`@RestControllerAdvice(basePackages = "com.bkurs.{domain}")` 로 본인 도메인만. 예:

```java
@RestControllerAdvice(basePackages = "com.bkurs.academy")
@RequiredArgsConstructor
public class AcademyExceptionHandler {
    private final ApiEnvelopes envelopes;

    @ExceptionHandler(AcademyNotFoundException.class)
    public ResponseEntity<ApiEnvelope<Void>> handle(AcademyNotFoundException ex, HttpServletRequest req) {
        ApiError error = ApiError.of(ex.errorCode(), ex.getMessage(), null, requestId(req));
        return envelopes.fail(ex.errorCode().status(), error, req);
    }
}
```

> envelope migration 미완료 도메인은 raw shape 반환 — `Exception.class` fallback 없는 이유 ([[envelope-migration]] 참조).

## 5. `web/` — 요청 ID + MDC

### 5.1 `RequestIdFilter`

- 요청에 UUID 부여 → `request.setAttribute(RequestIds.REQUEST_ID_ATTR, uuid)`
- `MDC.put("requestId", uuid)` — Logback `%X{requestId}` 패턴에서 출력
- 응답 header `X-Request-Id: <uuid>` 동봉
- 비동기 (`@Async`) 로 전파는 `AsyncConfig.TaskDecorator` 가.

### 5.2 `RequestIds` (utility)

- `RequestIds.REQUEST_ID_ATTR` = "requestId" 상수
- envelope helper 가 `request.getAttribute(...)` 로 추출

## 6. `messaging/` — 메시지 큐 추상화

### 6.1 `MessagingPublisher` (interface)

```java
public interface MessagingPublisher {
    String providerName();
    void publish(MessageTopic topic, Object payload);
}
```

### 6.2 구현

- `InMemoryMessagingPublisher` — default, log 만. test 도 동일.
- `RabbitMessagingPublisher` — `@ConditionalOnProperty bkurs.messaging.provider=rabbit`. `RabbitConfig` 가 exchange / queue / binding bean.

### 6.3 `MessageTopic` (enum)

현재 topic:

- `PAYMENT_OVERDUE` — batch 가 publish.
- (후속: NOTIFICATION_REQUESTED, RECORD_SYNCED ...)

> 새 topic 추가 시 본 enum + 본 표 갱신.

## 7. `config/` — Spring 설정

### 7.1 `SecurityConfig`

자세히 [[auth-and-security]]. 핵심: stateless + JWT Bearer + permitAll (signup/login/refresh/logout/health/swagger), 나머지 authenticated. `@EnableMethodSecurity` 로 `@PreAuthorize`.

### 7.2 `CorsConfig`

자세히 [[auth-and-security#cors]]. `cors.allowedOriginPatterns` (NOT origins) + `allowCredentials=true`. cookie refresh 가능.

### 7.3 `AsyncConfig`

2 thread pool:

- `externalCallExecutor` — gateway (Claude/STT/SendGrid …) 호출. IO bound 1~60s.
- `defaultAsyncExecutor` — event listener / light compute.

`TaskDecorator` 가 MDC (`requestId`) 전파 → 비동기 로그에서도 동일 requestId 매칭.

### 7.4 `OpenApiConfig`

Swagger 8 group 정의: auth / academy / course / record / plan / saenggibu / search / common (한국어). `/swagger-ui.html` 진입.

### 7.5 `TimeConfig`

`Clock.systemDefaultZone()` bean. envelope timestamp / academic period / refresh expiry 가 의존. 테스트 시 `@MockBean Clock` 으로 고정.

## 8. `health/`

`HealthController` (`/api/v1/health`) — smoke. permitAll. CI 가 startup 검증 시 호출.

actuator (`/actuator/health` `/actuator/info`) 도 노출 — Kubernetes liveness/readiness probe 호환.

## 9. 신규 도메인 추가 시 common 사용 체크리스트

1. `@RestControllerAdvice(basePackages = "com.bkurs.{newDomain}")` 작성 — domain BusinessException 매핑.
2. ErrorCode prefix 정하고 [[#2.3]] 표 갱신.
3. controller 에서 `ApiEnvelopes` 주입, 응답은 `envelopes.ok / created / noContent`.
4. DTO request 는 `@Valid` + `@NotBlank @Max @Email` 등 — `GlobalValidationExceptionHandler` 가 자동 envelope.
5. 비동기 호출은 `@Async("externalCallExecutor")`. MDC 전파 자동.
6. event publish 는 `ApplicationEventPublisher.publishEvent(...)` (in-process) 또는 `MessagingPublisher.publish(topic, payload)` (큐).

## cross-link

- [[../bkurs-be]]
- [[overview]]
- [[auth-and-security]]
- [[external-gateways]]
- [[envelope-migration]] — envelope 마이그레이션 진행
- [[async-mq-batch]] — messaging / async / batch SSoT
