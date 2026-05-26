---
title: "04. Controller + DTO + Bean Validation + 예외"
kind: guide
status: current
created_at: 2026-05-27T10:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, controller, validation, exception]
---

# 04. Controller + DTO + Bean Validation + 예외

**[[guidelines|↑ Hub]]** ← [[03-domain-and-jpa]] · → 다음 [[05-security-and-deployment]]

> `@RestController` + DTO (data class) + Bean Validation (`@Valid`) + 글로벌 예외 처리.

---

## 1. 응답 envelope — `ApiResponse<T>`

모든 응답이 일관된 모양:

```json
{
  "success": true,
  "message": "성공",
  "data": { ... }
}

// 실패
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력 형식이 올바르지 않습니다.",
    "details": { "email": "must be a valid email" }
  }
}
```

```kotlin
// src/main/kotlin/com/example/memo/common/ApiResponse.kt
package com.example.memo.common

import com.fasterxml.jackson.annotation.JsonInclude

@JsonInclude(JsonInclude.Include.NON_NULL)
data class ApiResponse<T>(
    val success: Boolean,
    val message: String? = null,
    val data: T? = null,
    val error: ApiError? = null,
) {
    companion object {
        fun <T> ok(data: T? = null, message: String = "성공"): ApiResponse<T> =
            ApiResponse(success = true, message = message, data = data)

        fun fail(code: String, message: String, details: Any? = null): ApiResponse<Nothing> =
            ApiResponse(success = false, error = ApiError(code, message, details))
    }
}

data class ApiError(
    val code: String,
    val message: String,
    val details: Any? = null,
)
```

> **`@JsonInclude(NON_NULL)`** — null 필드는 JSON 에서 제외.
> **`data class` companion factory** — `ApiResponse.ok(data)` / `ApiResponse.fail(...)`.

---

## 2. BusinessException — 도메인 예외

```kotlin
// src/main/kotlin/com/example/memo/common/BusinessException.kt
package com.example.memo.common

open class BusinessException(
    val status: Int,
    val code: String,
    message: String,
) : RuntimeException(message)


class NotFoundException(what: String = "리소스") : BusinessException(
    status = 404, code = "NOT_FOUND", message = "${what}를 찾을 수 없습니다.",
)

class ForbiddenException(message: String = "권한이 없습니다.") : BusinessException(
    status = 403, code = "FORBIDDEN", message = message,
)

class UnauthorizedException(message: String = "인증이 필요합니다.") : BusinessException(
    status = 401, code = "UNAUTHORIZED", message = message,
)

class ConflictException(message: String = "이미 존재합니다.") : BusinessException(
    status = 409, code = "CONFLICT", message = message,
)
```

사용:

```kotlin
val memo = memoRepository.findById(id).orElse(null)
    ?: throw NotFoundException("메모")
```

---

## 3. 글로벌 예외 처리 — `@RestControllerAdvice`

```kotlin
// src/main/kotlin/com/example/memo/common/ApiExceptionHandler.kt
package com.example.memo.common

import org.slf4j.LoggerFactory
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ApiExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(BusinessException::class)
    fun handleBusiness(ex: BusinessException): ResponseEntity<ApiResponse<Nothing>> {
        if (ex.status >= 500) log.error("BIZ 5xx: {}", ex.message, ex)
        else                  log.warn("BIZ {}: {}", ex.code, ex.message)

        return ResponseEntity
            .status(ex.status)
            .body(ApiResponse.fail(ex.code, ex.message ?: "error"))
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<ApiResponse<Nothing>> {
        val details = ex.bindingResult.fieldErrors.associate {
            it.field to (it.defaultMessage ?: "invalid")
        }
        log.warn("VALIDATION: {}", details)

        return ResponseEntity
            .status(422)
            .body(ApiResponse.fail("VALIDATION_ERROR", "입력 형식이 올바르지 않습니다.", details))
    }

    @ExceptionHandler(Exception::class)
    fun handleAny(ex: Exception): ResponseEntity<ApiResponse<Nothing>> {
        log.error("UNCAUGHT", ex)
        return ResponseEntity
            .status(500)
            .body(ApiResponse.fail("INTERNAL_SERVER_ERROR", "서버 내부 오류가 발생했습니다."))
    }
}
```

> **`@RestControllerAdvice`** = 모든 `@RestController` 의 예외를 catch.
> 순서: 구체적 (`BusinessException`) → 일반 (`Exception`).

---

## 4. DTO — Request / Response

### 4.1 Request DTO + Bean Validation

```kotlin
// src/main/kotlin/com/example/memo/memo/CreateMemoRequest.kt
package com.example.memo.memo

import jakarta.validation.constraints.Size

data class CreateMemoRequest(
    @field:Size(min = 1, max = 200, message = "제목은 1~200자")
    val title: String,

    @field:Size(max = 10_000, message = "본문은 10000자 이하")
    val content: String = "",
)


data class UpdateMemoRequest(
    @field:Size(min = 1, max = 200)
    val title: String? = null,

    @field:Size(max = 10_000)
    val content: String? = null,
)
```

> **`@field:Size(...)`** — Kotlin annotation target. Java 의 `@Size` 만 쓰면 어디에 붙는지 모호 → **반드시 `@field:` 명시**.
>
> 자주 쓰는 validation:
> - `@field:NotBlank` — 빈 문자열 / null X
> - `@field:NotNull` — null X
> - `@field:Size(min = 1, max = 100)` — 문자열 / 컬렉션 크기
> - `@field:Email` — email 형식
> - `@field:Min(1)` / `@field:Max(99)` — 숫자 범위
> - `@field:Pattern(regexp = "...")` — 정규식
> - `@field:Past` / `@field:Future` — 날짜

### 4.2 Response DTO

```kotlin
// src/main/kotlin/com/example/memo/memo/MemoResponse.kt
package com.example.memo.memo

import com.fasterxml.jackson.annotation.JsonProperty
import java.time.Instant

data class MemoResponse(
    val id: Long,
    val title: String,
    val content: String,

    @JsonProperty("createdAt")               // JSON field name 명시 (default snake_case 가 아니면 생략)
    val createdAt: Instant,
    val updatedAt: Instant,
) {
    companion object {
        fun from(memo: Memo): MemoResponse = MemoResponse(
            id = memo.id!!,                   // entity 가 saved 후라 non-null 보장
            title = memo.title,
            content = memo.content,
            createdAt = memo.createdAt,
            updatedAt = memo.updatedAt,
        )
    }
}
```

> **Entity → DTO mapping**: companion `from(...)` factory. 외부에 entity 직접 노출 X (lazy / cycle 위험).

---

## 5. Controller

```kotlin
// src/main/kotlin/com/example/memo/memo/MemoController.kt
package com.example.memo.memo

import com.example.memo.common.ApiResponse
import com.example.memo.user.User
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/memos")
class MemoController(
    private val memoService: MemoService,
) {

    @GetMapping
    fun list(
        @AuthenticationPrincipal user: User,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
    ): ApiResponse<List<MemoResponse>> {
        val memos = memoService.listByOwner(user.id!!, page, size)
        return ApiResponse.ok(memos.map(MemoResponse::from), "조회 성공")
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(
        @Valid @RequestBody req: CreateMemoRequest,
        @AuthenticationPrincipal user: User,
    ): ApiResponse<MemoResponse> {
        val memo = memoService.create(user, req.title, req.content)
        return ApiResponse.ok(MemoResponse.from(memo), "생성 완료")
    }

    @GetMapping("/{id}")
    fun get(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: User,
    ): ApiResponse<MemoResponse> {
        val memo = memoService.getOwned(id, user.id!!)
        return ApiResponse.ok(MemoResponse.from(memo))
    }

    @PatchMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody req: UpdateMemoRequest,
        @AuthenticationPrincipal user: User,
    ): ApiResponse<MemoResponse> {
        val memo = memoService.update(id, user.id!!, req.title, req.content)
        return ApiResponse.ok(MemoResponse.from(memo), "수정 완료")
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(
        @PathVariable id: Long,
        @AuthenticationPrincipal user: User,
    ) {
        memoService.delete(id, user.id!!)
    }
}
```

> **`@Valid`** = Spring 이 Bean Validation 실행. 실패 시 `MethodArgumentNotValidException` → 글로벌 핸들러 (§3) 가 422 응답.
> **`@AuthenticationPrincipal user: User`** = 현재 로그인된 사용자 (Security 가 SecurityContext 에 넣어둠). 다음 노트 [[05-security-and-deployment]] 에서 구현.

---

## 6. 호출 시나리오 — 응답 예시

```bash
# 1. 메모 작성 (정상)
curl -X POST http://localhost:8080/api/v1/memos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕하세요"}'

# 응답:
# 201 Created
# {
#   "success": true,
#   "message": "생성 완료",
#   "data": {
#     "id": 1,
#     "title": "첫 메모",
#     "content": "안녕하세요",
#     "createdAt": "2026-05-27T01:23:45.678Z",
#     "updatedAt": "2026-05-27T01:23:45.678Z"
#   }
# }


# 2. validation 실패 — title 누락
curl -X POST http://localhost:8080/api/v1/memos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"content":"title 없음"}'

# 응답:
# 422 Unprocessable Entity
# {
#   "success": false,
#   "error": {
#     "code": "VALIDATION_ERROR",
#     "message": "입력 형식이 올바르지 않습니다.",
#     "details": { "title": "제목은 1~200자" }
#   }
# }


# 3. 다른 user 의 메모 접근 — 403
curl http://localhost:8080/api/v1/memos/999 \
  -H "Authorization: Bearer <other_token>"

# 응답:
# 403 Forbidden
# {
#   "success": false,
#   "error": {
#     "code": "FORBIDDEN",
#     "message": "본인의 메모만 조회 가능합니다."
#   }
# }
```

---

## 7. Service Layer 정리 (다시)

[[03-domain-and-jpa#7 Service|03 §7]] 의 `MemoService` 를 그대로 사용. Controller 가 service 를 호출하고, service 가 Repository 와 트랜잭션 처리.

**계층 책임**:

| 계층 | 책임 |
| --- | --- |
| Controller | HTTP 입출력, DTO ↔ domain 변환, validation 트리거 |
| Service | 트랜잭션, business rule, 권한 검증, repository 조합 |
| Repository | DB 호출만 (CRUD) |
| Entity | 도메인 행동 (rich domain) |

→ Controller 는 절대 Repository 직접 호출 X. 항상 Service 통해.

---

## 8. 추가 패턴

### 8.1 paging — Spring Data Pageable

```kotlin
@GetMapping
fun list(
    @PageableDefault(size = 20, sort = ["createdAt"], direction = Sort.Direction.DESC)
    pageable: Pageable,
    @AuthenticationPrincipal user: User,
): ApiResponse<Page<MemoResponse>> {
    val page = memoRepository.findByOwnerId(user.id!!, pageable)
    return ApiResponse.ok(page.map(MemoResponse::from))
}
```

→ query: `?page=0&size=20&sort=createdAt,desc`. Spring 이 자동 parsing.

### 8.2 Idempotency — 같은 요청 두 번 처리 안 함

```kotlin
@PostMapping
fun create(
    @RequestHeader("X-Idempotency-Key", required = false) key: String?,
    @Valid @RequestBody req: CreateMemoRequest,
): ApiResponse<MemoResponse> {
    if (key != null) {
        idempotencyService.findByKey(key)?.let {
            return ApiResponse.ok(it, "이미 처리됨")        // 같은 응답 재반환
        }
    }
    val result = memoService.create(...)
    if (key != null) idempotencyService.save(key, result)
    return ApiResponse.ok(result)
}
```

### 8.3 부분 update 의 `null` vs `없음` 구분

PATCH 시 `null` 의도가 "필드 안 보냄" 이 아니라 "null 로 설정" 일 수도. Jackson 의 `JsonNullable` 또는 별도 `Optional` 사용:

```kotlin
data class UpdateMemoRequest(
    val title: String? = null,           // 안 보내면 null — 변경 X
    val content: String? = null,
)

// Service 에서
fun update(memoId: Long, ownerId: Long, req: UpdateMemoRequest): Memo {
    val memo = getOwned(memoId, ownerId)
    req.title?.let { memo.title = it }
    req.content?.let { memo.content = it }
    return memo
}
```

→ "null = 변경 X" 의도. 진짜 null 로 설정하려면 별도 sentinel 또는 `JsonNullable`.

---

## 9. 자주 막히는 함정 (Controller / Validation)

### 함정 1 — `@Valid` 누락
DTO 에 `@field:Size` 붙였지만 `@Valid @RequestBody` 안 쓰면 validation 동작 X.

### 함정 2 — `@Size` (target 없이)
data class 파라미터에 `@Size` 만 = 어디에 붙는지 모호. **`@field:Size`** 명시.

### 함정 3 — Entity 를 Response 직접 반환
lazy proxy / cycle / 내부 필드 노출. **DTO mapping**.

### 함정 4 — `@AuthenticationPrincipal` 의 type 불일치
SecurityConfig 에서 setAuthentication 한 principal 의 type 과 일치해야. 안 그러면 null.

### 함정 5 — `MethodArgumentNotValidException` 가 글로벌 핸들러 못 잡음
`@RestControllerAdvice` 가 아니라 `@ControllerAdvice` 면 (다른 advice) — 우리는 `@RestControllerAdvice`.

### 함정 6 — `404` 대신 `500` 응답
Service 가 `findById(...).get()` (Optional 비어있으면 NSEE) → 500. **`orElseThrow { NotFoundException(...) }`**.

### 함정 7 — `@RequestBody` 안 붙임
default 는 query parameter 로 인식. body 필요한 POST/PATCH 는 반드시 `@RequestBody`.

### 함정 8 — `@ResponseStatus(HttpStatus.CREATED)` 누락
POST 응답이 200 으로 — 표준은 201. 명시.

### 함정 9 — `private val memoService` 가 Kotlin 컴파일러에 visible 하지만 Spring 이 못 봄?
`@Autowired` 없이 constructor 만 — Spring 이 자동 주입 (constructor injection). visibility 무관. 단 final field 라 권장.

### 함정 10 — `companion object` 의 factory method 가 Java 에서 호출 시 `Companion.from(...)`
Java interop 필요 시 `@JvmStatic` 추가.

---

## 10. 다음

✅ Controller + DTO + Validation + 예외 끝.

→ [[05-security-and-deployment]] — Spring Security + JWT 발급 / 검증 + Docker 배포
