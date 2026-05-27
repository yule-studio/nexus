---
title: "표준 응답 envelope + 글로벌 예외 처리 (Kotlin)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T10:00:00+09:00
tags:
  - backend
  - kotlin-spring
  - common
  - response
  - exception
  - production
---

# 표준 응답 envelope + 글로벌 예외 처리 (Kotlin)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin idiom (data class / sealed / scope fn / extension) 적용 |

**[[../kotlin-spring|↑ Kotlin Spring]]** · **[[../../java-spring/common/response-envelope|↗ Java 원본]]**

> **이 문서가 본거지.** 모든 레시피 (`api-design/`) 는 여기 정의된 `CommonResponse` / `ResponseCode` / `BusinessException` / `ApiExceptionHandler` 를 그대로 사용.
> Java 원본과 동일한 외부 계약 (JSON envelope, code 체계) 을 유지하면서 Kotlin 관용구로 재작성.

---

## 1. 왜 envelope 통일이 첫 작업인가

- **클라이언트 (앱/웹) 가 한 가지 응답 모양만 처리** — 성공 / 실패 분기 단순.
- **에러 코드가 안정적** (`OK_001`, `BADREQ_002`, `UNAUTH_001` 같은 stable string) — 메시지 번역 / 분기 / 알람.
- **글로벌 예외 한 곳에 모음** — Controller 마다 try-catch 안 함.
- **운영 로그** 와 API 응답 코드가 1:1 — 장애 시 클라 측 코드만 보고 위치 추적.

---

## 2. 4 컴포넌트 한눈에

```
[Service] ── throw BusinessException(ResponseCode.X, "메시지") ──┐
                                                                 │
[Controller] ── return CommonResponse<T> ──────────────────┐     │
                                                            ▼     ▼
                                                ┌─────────────────┐
                                                │ ApiExceptionHandler  (@RestControllerAdvice)
                                                │   - BusinessException → CommonResponse(코드+메시지)
                                                │   - MethodArgumentNotValidException → 422
                                                │   - S3Exception / IOException / ... → 매핑
                                                │   - Exception (캐치올) → 500
                                                └─────────────────┘
                                                       │
                                                       ▼
                                            CommonResponse.toResponseEntity()
                                                       │
                                                       ▼
                                        ResponseEntity<CommonResponse>
```

---

## 3. `CommonResponse<T>` — 응답 envelope

```kotlin
// src/main/kotlin/com/example/shop/common/model/dto/response/CommonResponse.kt
@JsonInclude(JsonInclude.Include.NON_NULL)
data class CommonResponse<T>(
    val code: String,            // "OK_001" / "BADREQ_002" — stable string
    val status: ResponseCode,    // enum (HttpStatus 포함)
    val message: String,         // 사람이 읽는 메시지 (i18n 가능)
    val result: T? = null,       // 성공 시 데이터. null 이면 직렬화 제외.
) {
    companion object {
        fun <T> success(code: ResponseCode, message: String): CommonResponse<T> =
            CommonResponse(code.code, code, message)

        fun <T> success(code: ResponseCode, data: T, message: String): CommonResponse<T> =
            CommonResponse(code.code, code, message, data)

        fun <T> fail(code: ResponseCode, customMessage: String? = null): CommonResponse<T> =
            CommonResponse(code.code, code, customMessage ?: code.message)
    }
}

/** envelope → ResponseEntity 변환을 한 줄로 */
fun <T> CommonResponse<T>.toResponseEntity(): ResponseEntity<CommonResponse<T>> =
    ResponseEntity.status(status.httpStatus).body(this)
```

> **Kotlin idiom 포인트**:
> - `data class` 로 `equals` / `hashCode` / `copy` / `toString` 자동 생성 — Lombok `@Data`/`@Builder`/`@NoArgsConstructor`/`@AllArgsConstructor` 가 한 줄에.
> - factory 는 `companion object` — `CommonResponse.success(...)` 로 호출 (Java 와 동일한 사용감).
> - `customMessage: String? = null` — default parameter 로 overload 합칠 수 있음.
> - **extension function** `toResponseEntity()` — 도메인 외부 (controller) 에서 변환을 한 줄로.

### 3.1 응답 예시

성공:
```json
{ "code": "OK_001", "status": "OK", "message": "회원가입 성공" }
```

성공 + 데이터:
```json
{
  "code": "OK_001",
  "status": "OK",
  "message": "로그인 성공",
  "result": { "accessToken": "eyJ...", "refreshToken": "eyJ..." }
}
```

실패:
```json
{ "code": "UNAUTH_001", "status": "UNAUTHORIZED", "message": "입력하신 이메일과 비밀번호를 확인해 주세요." }
```

---

## 4. `ResponseCode` enum — 코드 / 메시지 / HTTP status 매핑

```kotlin
// src/main/kotlin/com/example/shop/common/model/enums/ResponseCode.kt
enum class ResponseCode(
    val httpStatus: HttpStatus,
    val code: String,
    val message: String,
) {

    /* ===== 2xx 성공 ===== */
    OK              (HttpStatus.OK,             "OK_001", "성공"),
    OK_EMPTY        (HttpStatus.OK,             "OK_002", "성공 + 데이터 없음"),

    /* ===== 4xx 클라이언트 ===== */
    // 400 BAD_REQUEST
    BAD_REQUEST             (HttpStatus.BAD_REQUEST, "BADREQ_001", "잘못된 요청입니다."),
    INVALID_INPUT_FORMAT    (HttpStatus.BAD_REQUEST, "BADREQ_002", "입력 형식이 올바르지 않습니다."),
    MISSING_REQUIRED_FIELD  (HttpStatus.BAD_REQUEST, "BADREQ_003", "필수 입력값이 누락되었습니다."),
    DUPLICATE_DATA          (HttpStatus.BAD_REQUEST, "BADREQ_004", "이미 존재하는 데이터입니다."),
    // 401 UNAUTHORIZED
    UNAUTHORIZED            (HttpStatus.UNAUTHORIZED, "UNAUTH_001", "인증이 필요합니다."),
    INVALID_TOKEN           (HttpStatus.UNAUTHORIZED, "UNAUTH_002", "유효하지 않은 토큰입니다."),
    // 403 FORBIDDEN
    FORBIDDEN               (HttpStatus.FORBIDDEN, "FORBID_001", "권한이 없습니다."),
    INSUFFICIENT_PERMISSION (HttpStatus.FORBIDDEN, "FORBID_002", "해당 리소스에 접근할 권한이 없습니다."),
    // 404 NOT_FOUND
    NOT_FOUND               (HttpStatus.NOT_FOUND, "NOTFND_001", "요청한 리소스를 찾을 수 없습니다."),
    USER_NOT_FOUND          (HttpStatus.NOT_FOUND, "NOTFND_002", "사용자를 찾을 수 없습니다."),
    // 409 CONFLICT (개선: "duplicate / 이미 존재"는 409 가 표준)
    CONFLICT                (HttpStatus.CONFLICT, "CONFLT_001", "리소스 상태 충돌"),
    // 410 GONE
    EXPIRED_AUTH_CODE       (HttpStatus.GONE, "GONE_001", "인증 코드가 만료되었습니다."),
    DELETED_RESOURCE        (HttpStatus.GONE, "GONE_002", "이미 삭제된 리소스입니다."),
    USER_ALREADY_DELETED    (HttpStatus.GONE, "GONE_003", "이미 삭제된 사용자입니다."),
    // 422 UNPROCESSABLE_ENTITY
    AUTH_UNVERIFIED         (HttpStatus.UNPROCESSABLE_ENTITY, "UNPROC_001", "인증이 완료되지 않았습니다."),
    ADDITIONAL_INFO_REQUIRED(HttpStatus.UNPROCESSABLE_ENTITY, "UNPROC_002", "추가 정보 입력이 필요합니다."),
    SOCIAL_EMAIL_MISSING    (HttpStatus.UNPROCESSABLE_ENTITY, "UNPROC_003", "소셜 로그인에서 이메일이 제공되지 않았습니다."),
    // 429 TOO_MANY_REQUESTS
    RATE_LIMIT_EXCEEDED     (HttpStatus.TOO_MANY_REQUESTS, "RATE_001", "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요."),

    /* ===== 5xx 서버 ===== */
    INTERNAL_SERVER_ERROR   (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_001", "서버 내부 오류"),
    DATABASE_ERROR          (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_002", "데이터베이스 오류"),
    EXTERNAL_API_ERROR      (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_003", "외부 API 호출 중 오류"),
    FILE_PROCESSING_ERROR   (HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERR_004", "파일 처리 오류"),
    ;

    /** 5xx 여부 — handler 에서 log level 분기에 사용 */
    fun is5xx(): Boolean = httpStatus.is5xxServerError
}
```

> **Kotlin idiom 포인트**:
> - Java 의 `private final` 필드 + getter 4 줄 → 생성자 파라미터에 `val` 만으로 끝.
> - `enum class` 의 멤버 함수 (`is5xx()`) 로 도메인 로직을 enum 안에 묶음 — 외부에서 `code.is5xx()` 로 자연스럽게.

### 4.1 개선 사항

| 원본 (job-answer-be) | 개선 |
| --- | --- |
| `DUPLICATE_DATA` 가 400 | **400 + 별도 409 `CONFLICT`** (REST 표준) |
| `SUFFICIENT_NOT_PERMISSIONS` (오타) | `INSUFFICIENT_PERMISSION` |
| `DDITIONAL_INFO_REQUIRED` (오타) | `ADDITIONAL_INFO_REQUIRED` |
| `APPLE_EMAIL_NOT_PROVIDED` 너무 구체적 | 일반화 `SOCIAL_EMAIL_MISSING` |
| 429 / rate limit 없음 | `RATE_LIMIT_EXCEEDED` 추가 |
| 한국어만 | 영문도 가능 (i18n 은 `message` 만 외부 번역) |

### 4.2 새 코드 추가 정책

- `<카테고리>_<번호 3자리>` — 예: `BADREQ_005`, `UNAUTH_003`
- 같은 도메인 코드는 같은 prefix
- 추가 시 PR 에 변경 이유 명시 / 클라이언트 측에 알림
- **삭제 / 의미 변경 금지** — 한 번 정의된 코드는 deprecated 만 (클라이언트 호환)

---

## 5. `BusinessException` — 도메인 예외

```kotlin
// src/main/kotlin/com/example/shop/common/exception/BusinessException.kt
class BusinessException(
    val responseCode: ResponseCode,
    val customMessage: String = responseCode.message,
) : RuntimeException(customMessage) {

    /** 코드 없이 메시지만 — 가급적 ResponseCode 명시 권장 */
    constructor(message: String) : this(ResponseCode.INTERNAL_SERVER_ERROR, message)

    fun is5xx(): Boolean = responseCode.is5xx()
}
```

> **Kotlin idiom 포인트**:
> - `customMessage: String = responseCode.message` — default parameter 로 Java 의 3 개 생성자 (코드만 / 코드+메시지 / 메시지만) 가 1 개 primary + 1 개 secondary 로 줄어듦.
> - `val` 로 자동 getter — Lombok `@Getter` 불필요.
> - `is5xx()` 는 `responseCode.is5xx()` 위임 — 한쪽이 SSoT (ResponseCode 안 메서드).

### 5.1 사용 예

```kotlin
// Service
if (userRepo.existsByEmailHash(emailHash)) {
    throw BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입된 이메일입니다.")
}

// 객체 못 찾음 — Kotlin null-safety 와 elvis 결합
return userRepo.findById(id)
    ?: throw BusinessException(ResponseCode.USER_NOT_FOUND)

// 권한 없음
if (!order.isOwnedBy(currentUserId)) {
    throw BusinessException(ResponseCode.FORBIDDEN, "본인의 주문만 조회 가능합니다.")
}
```

> **선택적 확장 — Optional 패턴 자체를 없애고 싶다면 extension 으로:**
> ```kotlin
> fun <T> T?.orThrow(code: ResponseCode, message: String? = null): T =
>     this ?: throw BusinessException(code, message ?: code.message)
>
> // 사용
> val user = userRepo.findById(id).orThrow(ResponseCode.USER_NOT_FOUND)
> ```

---

## 6. `ApiExceptionHandler` — 글로벌

```kotlin
// src/main/kotlin/com/example/shop/common/handler/ApiExceptionHandler.kt
@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
class ApiExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    companion object {
        /** 어떤 ApiLogFilter / 모니터링 filter 가 예외 객체를 가져갈 attribute key */
        const val EXCEPTION_ATTR = "apiLog.exception"
    }

    // -----------------------------------------------
    // 1) 도메인 예외 — BusinessException
    // -----------------------------------------------
    @ExceptionHandler(BusinessException::class)
    fun handleBusiness(ex: BusinessException, req: HttpServletRequest): ResponseEntity<CommonResponse<Unit>> {
        req.setAttribute(EXCEPTION_ATTR, ex)
        if (ex.is5xx()) log.error("BIZ 5xx: {}", ex.message, ex)
        else            log.warn("BIZ: code={}, msg={}", ex.responseCode.code, ex.message)

        return CommonResponse.fail<Unit>(ex.responseCode, ex.customMessage)
            .toResponseEntity()
    }

    // -----------------------------------------------
    // 2) Bean Validation — @Valid, @Validated
    // -----------------------------------------------
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(
        ex: MethodArgumentNotValidException,
        req: HttpServletRequest,
    ): ResponseEntity<CommonResponse<Map<String, String>>> {
        req.setAttribute(EXCEPTION_ATTR, ex)

        val fieldErrors = ex.bindingResult.fieldErrors.associate {
            it.field to (it.defaultMessage ?: "invalid")
        }
        log.warn("VALIDATION: fields={}", fieldErrors)

        val body = CommonResponse(
            code = ResponseCode.INVALID_INPUT_FORMAT.code,
            status = ResponseCode.INVALID_INPUT_FORMAT,
            message = "입력 형식이 올바르지 않습니다.",
            result = fieldErrors,
        )
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(body)
    }

    // -----------------------------------------------
    // 3) DB 무결성 (unique violation / FK)
    // -----------------------------------------------
    @ExceptionHandler(DataIntegrityViolationException::class)
    fun handleDataIntegrity(
        ex: DataIntegrityViolationException,
        req: HttpServletRequest,
    ): ResponseEntity<CommonResponse<Unit>> {
        req.setAttribute(EXCEPTION_ATTR, ex)
        log.warn("DB integrity: {}", ex.mostSpecificCause.message)

        return CommonResponse.fail<Unit>(ResponseCode.CONFLICT, "리소스 상태 충돌")
            .toResponseEntity()
    }

    // -----------------------------------------------
    // 4) JPA 낙관 락
    // -----------------------------------------------
    @ExceptionHandler(OptimisticLockingFailureException::class)
    fun handleOptimisticLock(
        ex: OptimisticLockingFailureException,
        req: HttpServletRequest,
    ): ResponseEntity<CommonResponse<Unit>> {
        req.setAttribute(EXCEPTION_ATTR, ex)
        log.warn("Optimistic lock: {}", ex.message)

        return CommonResponse.fail<Unit>(
            ResponseCode.CONFLICT,
            "동시 수정이 감지되었습니다. 다시 시도해 주세요.",
        ).toResponseEntity()
    }

    // -----------------------------------------------
    // 5) S3 / 파일 처리
    // -----------------------------------------------
    @ExceptionHandler(S3Exception::class)
    fun handleS3(ex: S3Exception, req: HttpServletRequest): ResponseEntity<CommonResponse<Unit>> {
        req.setAttribute(EXCEPTION_ATTR, ex)
        log.warn("S3 error: {}", ex.message)
        return CommonResponse.fail<Unit>(
            ResponseCode.EXTERNAL_API_ERROR,
            "S3 처리 중 오류가 발생했습니다.",
        ).toResponseEntity()
    }

    @ExceptionHandler(IOException::class)
    fun handleIO(ex: IOException, req: HttpServletRequest): ResponseEntity<CommonResponse<Unit>> {
        req.setAttribute(EXCEPTION_ATTR, ex)
        log.warn("IO error: {}", ex.message)
        return CommonResponse.fail<Unit>(
            ResponseCode.FILE_PROCESSING_ERROR,
            "파일 처리 중 오류가 발생했습니다.",
        ).toResponseEntity()
    }

    // -----------------------------------------------
    // 6) 캐치올 — Exception (마지막 안전망)
    // -----------------------------------------------
    @ExceptionHandler(Exception::class)
    fun handleAny(ex: Exception, req: HttpServletRequest): ResponseEntity<CommonResponse<Unit>> {
        req.setAttribute(EXCEPTION_ATTR, ex)
        log.error("UNCAUGHT: {}", ex.message, ex)

        // 운영에선 상세 메시지 노출 X (보안)
        return CommonResponse.fail<Unit>(
            ResponseCode.INTERNAL_SERVER_ERROR,
            "서버 내부 오류가 발생했습니다.",
        ).toResponseEntity()
    }
}
```

> **Kotlin idiom 포인트**:
> - `Map<String, String>` 만들 때 `stream().collect(Collectors.toMap(...))` → `associate { it.field to ... }` 단 한 줄.
> - `Void` → `Unit` (Kotlin 의 void 대응).
> - `ex.message` 가 `String?` 이므로 elvis `?:` 로 default — `if/else` 없이.
> - `ex.bindingResult.fieldErrors` — Java getter 가 property syntax 로 자동 변환.
> - `companion object` 의 `const val` — Java `public static final` 과 동등.

### 6.1 개선 사항

| 원본 | 개선 |
| --- | --- |
| `ex.toString()` 을 message 로 (5xx 에서) | 운영에선 일반 메시지, 디버그 메시지는 로그로만 |
| `MethodArgumentNotValidException` 의 details 없음 | `result` 필드에 field 별 에러 맵 |
| `DataIntegrityViolationException` 핸들러 없음 | 409 매핑 추가 |
| `OptimisticLockingFailureException` 핸들러 없음 | 409 + 재시도 안내 |
| 모든 5xx 로그 `warn` | 5xx 는 `error`, 4xx 는 `warn` |
| `ResponseUtil.getHttpStatus` 호출 | `toResponseEntity()` extension (간결) |

### 6.2 `@Order(HIGHEST_PRECEDENCE)` 의 의미

- 다른 `@RestControllerAdvice` 가 있어도 이게 먼저 발동
- Spring Security 의 `AccessDeniedHandler` 와는 별도 (Security 는 filter chain 단)

---

## 7. Controller 에서 사용 패턴

```kotlin
@PostMapping("/signup")
@SecurityRequirements()                          // 인증 면제
fun signup(
    @Valid @RequestBody req: SignupRequest,
): ResponseEntity<CommonResponse<SignupResponse>> {
    val data = signupService.signup(req)
    return CommonResponse.success(ResponseCode.OK, data, "회원가입 성공")
        .toResponseEntity()
}
```

또는 `ResponseEntity` 없이 (status 가 ResponseCode 와 동기화):

```kotlin
@PostMapping("/signup")
@ResponseStatus(HttpStatus.OK)                   // 명시
fun signup(@Valid @RequestBody req: SignupRequest): CommonResponse<SignupResponse> =
    CommonResponse.success(ResponseCode.OK, signupService.signup(req), "회원가입 성공")
```

> **권장**: 정상 응답은 `ResponseEntity` 없이 단일 expression body. 실패는 throw `BusinessException` → handler 가 status 매핑. Controller 가 간결.

---

## 8. `ResponseUtil` 가 사라진 이유

Java 원본에서는 `ResponseUtil.getHttpStatus(response)` 가 별도 헬퍼였지만, Kotlin 에서는 **extension function** `CommonResponse<T>.toResponseEntity()` 가 같은 역할을 type-safe 하게 수행. 별도 유틸 클래스 불필요.

```kotlin
// Java 호환이 필요하면 file-level util 로 노출
@file:JvmName("ResponseUtil")
package com.example.shop.common.util

fun getHttpStatus(response: CommonResponse<*>): HttpStatus =
    response.status.httpStatus
```

→ Java 코드에서 `ResponseUtil.getHttpStatus(response)` 로 호출 가능.

---

## 9. 로깅 / 감사 — ApiLogFilter 와 연동

`ApiExceptionHandler.EXCEPTION_ATTR` 로 예외 객체를 request attribute 에 저장. 별도 `ApiLogFilter` 가 응답 후 picked up:

```kotlin
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)            // Exception handler 보다 뒤
class ApiLogFilter(
    private val logs: ApiLogRepository,
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        req: HttpServletRequest,
        res: HttpServletResponse,
        chain: FilterChain,
    ) {
        val start = Instant.now()
        try {
            chain.doFilter(req, res)
        } finally {
            val ex = req.getAttribute(ApiExceptionHandler.EXCEPTION_ATTR) as? Exception
            logs.record(
                ApiLogRow(
                    method   = req.method,
                    path     = req.requestURI,
                    status   = res.status,
                    durMs    = Duration.between(start, Instant.now()).toMillis(),
                    exType   = ex?.javaClass?.simpleName,
                    exMsg    = ex?.message,
                ),
            )
        }
    }
}
```

> **Kotlin idiom 포인트**:
> - `as?` (safe cast) — Java 의 `instanceof` + cast 보다 안전.
> - `ex?.javaClass?.simpleName` / `ex?.message` — chain null-safety.
> - named arguments 로 `ApiLogRow` 생성자 호출 — 필드 순서 헷갈림 방지.

→ 모든 API 호출이 access log 보다 풍부한 메타 (예외 객체) 를 가짐. 운영 대시보드 / 알람 (5xx 비율) 에 활용.

---

## 10. 테스트

```kotlin
@Test
fun `business_exception_returns_correct_code_and_status`() {
    val ex = BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입")
    val res = handler.handleBusiness(ex, mockRequest())

    assertThat(res.statusCode.value()).isEqualTo(400)
    assertThat(res.body!!.code).isEqualTo("BADREQ_004")
    assertThat(res.body!!.message).isEqualTo("이미 가입")
}

@Test
fun `validation_returns_field_errors_in_result`() {
    val ex = mockValidationException(mapOf("email" to "must be email"))
    val res = handler.handleValidation(ex, mockRequest())

    assertThat(res.statusCode.value()).isEqualTo(422)
    assertThat(res.body!!.result).containsEntry("email", "must be email")
}

@Test
fun `uncaught_exception_does_not_leak_internal_message`() {
    val ex = RuntimeException("DB password = abc123")
    val res = handler.handleAny(ex, mockRequest())

    assertThat(res.body!!.message).isEqualTo("서버 내부 오류가 발생했습니다.")
    // 내부 메시지 노출 X
}
```

> **Kotlin idiom 포인트**:
> - 백틱으로 감싼 함수명 — JUnit 표시 시 한글/공백 사용 자유.
> - `mapOf("email" to "...")` — Java `Map.of(...)` 보다 짧음.
> - `res.body!!` — Kotlin 의 명시적 non-null assertion. 테스트에선 OK.

---

## 11. 운영 체크리스트

- [ ] 모든 Controller 에 try-catch 없음 — 예외는 handler 가 처리
- [ ] 모든 도메인 예외가 `BusinessException(ResponseCode.X, ...)` 형태
- [ ] `Exception` 캐치올은 운영에서 메시지 노출 X
- [ ] 5xx 비율 모니터링 (APM / Prometheus)
- [ ] 새 `ResponseCode` 추가 시 클라이언트 측에 알림
- [ ] `ApiLogFilter` 가 모든 응답을 기록
- [ ] `ApiExceptionHandler` 는 `@Order(HIGHEST_PRECEDENCE)`
- [ ] Spring Security 의 401/403 은 별도 (filter 단) — `JwtAuthenticationEntryPoint` ([[security-config]])

---

## 12. 함정 모음

### 함정 1 — `ex.toString()` 을 응답 메시지로
스택 / DB 패스워드 / 내부 path 노출. **응답에는 일반 메시지, 디버그는 로그**.

### 함정 2 — `responseCode` 이 nullable
Java 원본의 함정. Kotlin 에서는 primary constructor 에 non-null `ResponseCode` 강제 — 컴파일 단에서 차단됨.

### 함정 3 — Validation 의 details 없음
사용자가 어느 필드가 틀렸는지 모름. **`result` 필드에 field 별 에러**.

### 함정 4 — `DataIntegrityViolationException` 안 잡음
500 으로 빠짐. 실제로는 409 (unique / FK 충돌). **별도 핸들러**.

### 함정 5 — Spring Security 의 401/403 도 `@RestControllerAdvice` 가 처리할 거라 기대
filter chain 단에서 발동 — handler 가 못 잡음. **`AuthenticationEntryPoint`, `AccessDeniedHandler` 별도**.

### 함정 6 — `@Order` 없음
다른 advice 와 충돌 시 어떤 게 발동할지 모름. **`@Order(HIGHEST_PRECEDENCE)`**.

### 함정 7 — `BindingResult` 의 field 가 중복
같은 필드에 여러 에러가 붙음. `associate { ... }` 는 마지막 값으로 덮어쓰니, 첫 값 보존이 필요하면 `associateBy({ it.field }, { it.defaultMessage ?: "invalid" }) { a, _ -> a }` 또는 list 로.

### 함정 8 — `result` 에 sensitive data
예외 메시지에 카드번호 / 패스워드 등 들어가지 않게 검토.

### 함정 9 — 모든 응답에 `result` 강제
성공 응답에 `result: null` 표시 = 클라가 항상 `if (result)` 처리. **`@JsonInclude(NON_NULL)` + Kotlin default `null`**.

### 함정 10 — code 의 의미 변경
한 번 정의된 코드의 HTTP status / 의미 바뀌면 클라이언트 깨짐. **deprecated 만, 삭제 X**.

### 함정 11 — Kotlin 한정: `Unit` vs `Void`
`CommonResponse<Void>` 처럼 Java 식으로 쓰면 Kotlin 컴파일은 통과해도 Jackson 직렬화에서 어색. **`CommonResponse<Unit>` 사용**, `result: Unit?` 은 default `null` 이라 JSON 에서 빠짐.

### 함정 12 — Kotlin 한정: Lombok 없음 / 잘 안 됨
Kotlin 에선 Lombok 의 `@Data`/`@Builder` 가 무의미 (`data class` 가 대체). 만약 Java module 과 혼용해야 한다면 `kapt` 셋업 — 그러나 Kotlin only 라면 절대 도입하지 말 것.

---

## 13. 관련

- [[security-config]] (다음) — Security filter chain + Cors + JwtAuthenticationEntryPoint
- [[../api-docs/swagger-springdoc]] — `@ApiResponse` 에 `ResponseCode` 마다 응답 예시
- [[../api-design/api-design|↗ API 레시피]] — 본 패턴을 쓰는 8 레시피
- [[../pitfalls/transaction-pitfalls]] (예정)
- [[../../java-spring/common/response-envelope|↗ Java 원본]]
