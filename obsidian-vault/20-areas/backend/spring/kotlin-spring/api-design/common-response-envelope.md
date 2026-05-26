---
title: "공통 Response Envelope (ApiEnvelope) 표준 — Kotlin"
kind: guideline
area: backend/spring/kotlin-spring/api-design
status: standard
created_at: 2026-05-26T11:30:00+09:00
agent: engineering-agent/tech-lead
tags: [api-design, response, envelope, kotlin-spring, standard]
related:
  - "[[api-design]]"
  - "[[../common/response-envelope]]"
  - "[[../../java-spring/api-design/common-response-envelope]]"
home_hub: api-design
---

# 공통 Response Envelope 표준 — Kotlin

> 모든 REST API 응답을 단일 envelope 으로 감싸 FE / 다중 client 가 일관된
> 구조로 처리. **success/data/error/meta** 4 필드 고정.
> Java 원본 [[../../java-spring/api-design/common-response-envelope|↗ 여기]]
> 의 Kotlin 변형.

> **본 vault 의 본거지 envelope** 은 [[../common/response-envelope]] (`CommonResponse<T>`).
> 본 노트는 `bkurs-be` 스타일의 `ApiEnvelope<T>` 표준 — `success/data/error/meta` 4 필드 + `RequestId` filter 까지 정리한 변형.

## 1. 왜 envelope 인가

### 장점

- **FE 단일 처리 경로**: success 분기 1번 + error 분기 1번. 도메인 별 분기 X.
- **추적성**: meta.requestId 가 모든 응답 + 로그에 동일 → 장애 조사 5초.
- **버전 변경 영향 최소**: data 안 schema 만 바꿔도 envelope 외피는 stable.
- **error 표준화**: code (machine), messageKey (i18n), message (사람) 3 layer.
- **fieldErrors**: Bean Validation 결과를 form field 별 노출 → FE 즉시 UI 매핑.

### 단점 (vs raw entity 응답)

- 한 단계 더 깊어짐 (`body.data.X`).
- 모든 controller method signature 가 `ResponseEntity<ApiEnvelope<T>>`.

→ **trade-off 우위**: 단점 (depth 1) 보다 장점 (추적/일관/i18n) 이 훨씬 큼.

## 2. 표준 응답 shape

### 성공
```json
{
  "success": true,
  "data": { ... 도메인 응답 ... },
  "error": null,
  "meta": {
    "requestId": "req-a1b2c3d4",
    "timestamp": "2026-05-20T13:34:56.789Z"
  }
}
```

### 실패
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "COM-4003",
    "status": 400,
    "message": "입력값 검증에 실패했습니다.",
    "messageKey": "error.common.validation_error",
    "fieldErrors": [
      {"field": "email", "code": "NotBlank", "message": "이메일은 필수입니다"},
      {"field": "password", "code": "Size", "message": "8자 이상"}
    ],
    "requestId": "req-a1b2c3d4"
  },
  "meta": {
    "requestId": "req-a1b2c3d4",
    "timestamp": "2026-05-20T13:34:56.789Z"
  }
}
```

### 핵심 invariant

- `success=true` → `data != null && error == null`
- `success=false` → `data == null && error != null`
- `meta` 는 항상 존재 (성공/실패 무관).
- `204 No Content` 는 envelope 없이 빈 body (브라우저 호환).

## 3. 컴포넌트 구조

```
src/main/kotlin/com/{org}/{app}/common/
  ├── api/
  │   ├── ApiEnvelope.kt       // data class (success/data/error/meta + invariant)
  │   ├── ApiMeta.kt           // data class (requestId, timestamp)
  │   └── ApiEnvelopes.kt      // @Component (Clock 주입 + 헬퍼)
  ├── error/
  │   ├── ApiError.kt          // data class (code/status/message/messageKey/fieldErrors/requestId)
  │   ├── ErrorCodeSpec.kt     // interface (status/code/messageKey/defaultMessage)
  │   └── FieldErrorItem.kt    // data class (field/code/message)
  ├── enums/
  │   └── ErrorCode.kt         // enum class : ErrorCodeSpec (공통 COM-4xxx/5xxx)
  ├── exception/
  │   ├── BusinessException.kt        // sealed class 베이스
  │   ├── InvalidInputException.kt    // 400
  │   ├── ValidationException.kt      // 400 + fieldErrors
  │   ├── UnauthorizedException.kt    // 401
  │   ├── ForbiddenException.kt       // 403
  │   ├── NotFoundException.kt        // 404
  │   ├── ConflictException.kt        // 409
  │   └── ServiceUnavailableException.kt  // 503
  ├── handler/
  │   └── GlobalExceptionHandler.kt   // @RestControllerAdvice
  └── web/
      ├── RequestIds.kt        // X-Request-Id header 상수 (object)
      └── RequestIdFilter.kt   // OncePerRequestFilter — UUID 생성 + MDC 주입
```

> **Kotlin idiom 포인트**:
> - `BusinessException` 은 `sealed class` — 모든 도메인 예외가 한 파일 안 또는 같은 모듈 안에서 type-exhaustive. `when (ex) { is NotFoundException -> ... }` 가 컴파일러로 강제됨.
> - 상수 holder 는 `object RequestIds { const val ... }`.

## 4. ErrorCode 네이밍

`{도메인}-{HTTP status}{seq:3자리}` 형식.

| 예 | 의미 |
|---|---|
| `COM-4001` | Common, 400, #001 (INVALID_INPUT) |
| `COM-4003` | Common, 400, #003 (VALIDATION_ERROR) |
| `COM-5001` | Common, 500, #001 (INTERNAL_SERVER_ERROR) |
| `ACC-4040` | Account, 404, #0 (ACCOUNT_NOT_FOUND) |
| `PAY-4090` | Payment, 409, #0 (PAYMENT_DUPLICATE) |

- 도메인 prefix 는 3자 (영문 대문자) 권장.
- 같은 도메인 안 ErrorCode 는 1 enum.
- 공통 COM-* 는 envelope 패키지 default 제공.

## 5. Controller 사용 패턴

### 5-1. ok 응답
```kotlin
@RestController
@RequestMapping("/api/v1/students/me")
class StudentProfileController(
    private val service: StudentProfileService,
    private val envelopes: ApiEnvelopes,
) {

    @GetMapping
    fun get(
        @AuthenticationPrincipal accountId: Long,
        request: HttpServletRequest,
    ): ResponseEntity<ApiEnvelope<StudentProfileView>> =
        envelopes.ok(StudentProfileView.from(service.get(accountId)), request)
}
```

### 5-2. 201 Created
```kotlin
@PostMapping
fun create(/* ... */): ResponseEntity<ApiEnvelope<StudentProfileView>> {
    val saved = service.create(/* ... */)
    val location = URI.create("/api/v1/students/${saved.id}")
    return envelopes.created(location, StudentProfileView.from(saved), request)
}
```

### 5-3. 204 No Content
```kotlin
@DeleteMapping("/{id}")
fun delete(@PathVariable id: Long): ResponseEntity<Unit> {
    service.delete(id)
    return envelopes.noContent()  // envelope 없음, 표준 204
}
```

### 5-4. 도메인 예외
```kotlin
// service 단에서
if (notFound) throw NotFoundException(AccountErrorCode.ACCOUNT_NOT_FOUND, "id=$id")

// controller 는 try/catch 없음 — GlobalExceptionHandler 가 ApiError 로 변환
```

> **Kotlin idiom**: `Unit` 반환은 `ResponseEntity<Unit>` — Java `ResponseEntity<Void>` 와 동등.

## 6. GlobalExceptionHandler 패턴

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler(
    private val envelopes: ApiEnvelopes,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(BusinessException::class)
    fun handleBusiness(
        ex: BusinessException,
        request: HttpServletRequest,
    ): ResponseEntity<ApiEnvelope<Unit>> {
        val error = ApiError.of(ex.errorCode, ex.message, ex.fieldErrors, requestId(request))
        return envelopes.fail(ex.errorCode.status, error, request)
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(
        ex: MethodArgumentNotValidException,
        request: HttpServletRequest,
    ): ResponseEntity<ApiEnvelope<Unit>> {
        val fieldErrors = ex.bindingResult.fieldErrors.map {
            FieldErrorItem(it.field, it.code ?: "Invalid", it.defaultMessage.orEmpty())
        }
        val error = ApiError.of(ErrorCode.VALIDATION_ERROR, null, fieldErrors, requestId(request))
        return envelopes.fail(HttpStatus.BAD_REQUEST, error, request)
    }

    @ExceptionHandler(Exception::class)
    fun handleAny(ex: Exception, request: HttpServletRequest): ResponseEntity<ApiEnvelope<Unit>> {
        log.error("Unhandled exception", ex)
        val error = ApiError.of(
            ErrorCode.INTERNAL_SERVER_ERROR,
            "예상치 못한 오류",
            null,
            requestId(request),
        )
        return envelopes.fail(HttpStatus.INTERNAL_SERVER_ERROR, error, request)
    }

    private fun requestId(request: HttpServletRequest): String =
        request.getAttribute(RequestIds.REQUEST_ID_ATTR) as? String ?: ""
}
```

> **Kotlin idiom 포인트**:
> - `ex.bindingResult.fieldErrors.map { ... }` — Java stream + collect 보다 짧음.
> - `it.defaultMessage.orEmpty()` — `Optional.ofNullable(x).orElse("")` 한 줄.
> - `as? String ?: ""` — safe cast + elvis fallback.

## 7. RequestIdFilter

```kotlin
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
class RequestIdFilter : OncePerRequestFilter() {

    override fun doFilterInternal(
        req: HttpServletRequest,
        res: HttpServletResponse,
        chain: FilterChain,
    ) {
        val requestId = req.getHeader("X-Request-Id")
            ?.takeIf { it.isNotBlank() }
            ?: "req-${UUID.randomUUID().toString().substring(0, 8)}"

        req.setAttribute(RequestIds.REQUEST_ID_ATTR, requestId)
        res.setHeader("X-Request-Id", requestId)
        MDC.put("requestId", requestId)
        try {
            chain.doFilter(req, res)
        } finally {
            MDC.remove("requestId")
        }
    }
}
```

> **Kotlin idiom**: header → `?.takeIf { it.isNotBlank() } ?: default` — null-or-blank fallback 한 expression.

## 8. FE 처리 패턴

```typescript
// shared/api.ts
type ApiEnvelope<T> = {
  success: boolean;
  data: T | null;
  error: ApiError | null;
  meta: { requestId: string; timestamp: string };
};

type ApiError = {
  code: string;
  status: number;
  message: string;
  messageKey: string;
  fieldErrors: { field: string; code: string; message: string }[];
  requestId: string;
};

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(path, init);
  const envelope: ApiEnvelope<T> = await res.json();
  if (!envelope.success) {
    throw new ApiClientError(envelope.error!);
  }
  return envelope.data!;
}
```

## 9. 마이그레이션 전략 (기존 raw 응답 → envelope)

기존 프로젝트 적용 시:

1. **foundation 추가** — common/ 패키지 (api/error/enums/exception/handler/web).
2. **신규 endpoint 부터 envelope 강제** — 새 PR 의 controller 는 `ApiEnvelope<T>` 만.
3. **기존 endpoint 도메인 별 점진 migration** — domain unit (controller class) 단위 PR.
4. **FE 동시 migration** — domain unit 별로 BE+FE 함께. mixed 기간 최소화.
5. **회귀 보강** — response shape 검증 (`body.success`, `body.data.x`, `body.error.code`).
6. **migration 완료 시 강제**: lint / archUnit 룰로 `ResponseEntity<ApiEnvelope<T>>` 외 금지.

## 10. 안티 패턴

- ❌ raw entity 반환 (`return entity`) — envelope 우회.
- ❌ `Map<String, Any?>` 응답 — schema 불명.
- ❌ controller 에서 try/catch 후 직접 ResponseEntity.status(...).body(...) — handler 우회.
- ❌ 도메인 별 다른 error shape (어떤 endpoint 는 `{error: "..."}`, 다른 endpoint 는 `{message: "..."}`).
- ❌ 200 OK 인데 body 에 `{success: false}` — HTTP status 와 envelope.success 불일치.

### Kotlin 한정 안티 패턴

- ❌ `ApiEnvelope` 를 Java-style `class` 로 — `data class` 가 직렬화 / equals / copy 모두 자동.
- ❌ `sealed class BusinessException` 의 sub-class 를 다른 module 에서 추가 — sealed 의 핵심은 same-module 내 exhaustive. 다른 모듈에서 도메인 예외가 필요하면 본 베이스를 `open class` 로 풀거나 별도 sealed hierarchy 신설.

## 관련

- [[api-design]] — 상위 인덱스
- [[../common/response-envelope]] — 본 vault 의 본거지 envelope (`CommonResponse<T>`)
- [[../../java-spring/api-design/common-response-envelope|↗ Java 원본]]
- source: [codwithyc/spring-sandbox](https://github.com/codwithyc/spring-sandbox) common/* 패키지
