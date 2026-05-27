---
title: "07. Controller 작성 패턴 모음"
kind: guide
status: current
created_at: 2026-05-27T11:15:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, controller, patterns]
---

# 07. Controller 작성 패턴 모음

**[[guidelines|↑ Hub]]** ← [[06-swagger-openapi]] · → 다음 [[08-actuator-monitoring]]

> @RestController 의 다양한 request / response 패턴 정리. 04-controller-and-validation 보완.

---

## 1. 매핑 어노테이션 차이 — 한 표

| 어노테이션 | 어디서 | 예시 |
| --- | --- | --- |
| `@PathVariable` | URL path 의 일부 | `/users/{id}` → `id: Long` |
| `@RequestParam` | URL query string | `?page=0&size=20` → `page: Int` |
| `@RequestBody` | HTTP body (JSON) | `{"title":"x"}` → `req: CreateRequest` |
| `@RequestHeader` | HTTP header | `Authorization: Bearer ...` → `auth: String` |
| `@CookieValue` | Cookie | `JSESSIONID=...` → `sessionId: String` |
| `@ModelAttribute` | form data 또는 multi-part | (form / file upload) |
| `@AuthenticationPrincipal` | SecurityContext 의 user | → `user: User` |

---

## 2. `@PathVariable` — URL path

```kotlin
// 단일
@GetMapping("/users/{id}")
fun get(@PathVariable id: Long): UserResponse = ...

// 여러 개
@GetMapping("/users/{userId}/memos/{memoId}")
fun getMemo(
    @PathVariable userId: Long,
    @PathVariable memoId: Long,
): MemoResponse = ...

// 이름 다를 때
@GetMapping("/users/{id}")
fun get(@PathVariable("id") userId: Long): UserResponse = ...

// optional path — 잘 안 씀, 두 endpoint 로 분리 권장
@GetMapping("/users", "/users/{id}")
fun getOrList(@PathVariable(required = false) id: Long?): Any = ...
```

> **Kotlin 한정**: parameter name 보존 위해 `@PathVariable("id")` 명시 권장. 또는 build.gradle.kts 의 `-java-parameters` (kotlin("plugin.spring") 적용 시 자동).

---

## 3. `@RequestParam` — query string

```kotlin
@GetMapping("/users")
fun list(
    @RequestParam(defaultValue = "0") page: Int,             // ?page=
    @RequestParam(defaultValue = "20") size: Int,
    @RequestParam(required = false) keyword: String?,        // optional
    @RequestParam(name = "q", required = false) query: String?,  // 다른 이름
    @RequestParam status: List<String>,                       // ?status=A&status=B
    @RequestParam tags: Set<String>,                          // ?tags=t1,t2
    @RequestParam params: Map<String, String>,                 // 모든 query 를 dict 로
): UserListResponse = ...
```

> **`Map<String, String>`** 으로 받으면 모든 query parameter 가 한 번에. 자유로운 query 처리 시 유용.

### 3.1 enum + validation

```kotlin
enum class SortBy { LATEST, OLDEST, NAME }

@GetMapping("/users")
fun list(
    @RequestParam(defaultValue = "LATEST") sort: SortBy,
    @RequestParam(defaultValue = "20") @Min(1) @Max(100) size: Int,
): UserListResponse = ...
```

> Spring 이 string → enum 자동 변환 + `@Min`/`@Max` 검증. 잘못된 enum 값 → 400.

### 3.2 Pageable — Spring Data 표준

```kotlin
@GetMapping("/users")
fun list(
    @PageableDefault(size = 20, sort = ["createdAt"], direction = Sort.Direction.DESC)
    pageable: Pageable,
): Page<UserResponse> = userRepository.findAll(pageable).map(UserResponse::from)

// 호출: ?page=0&size=20&sort=createdAt,desc
```

→ Spring 이 query parameter → `Pageable` 자동 변환. Repository 가 `Pageable` 받음.

---

## 4. `@RequestBody` — JSON body

```kotlin
@PostMapping("/users")
fun create(@Valid @RequestBody req: CreateUserRequest): UserResponse = ...
```

`Content-Type: application/json` 이고 body 가 JSON 이면 Jackson 이 자동 deserialization.

### 4.1 다중 body 불가능

```kotlin
// ❌ 하나의 endpoint 는 body 1개만
fun create(@RequestBody a: A, @RequestBody b: B) = ...

// ✅ wrapping object
data class CreateRequest(val a: A, val b: B)
fun create(@RequestBody req: CreateRequest) = ...
```

### 4.2 body 가 비어있을 수도 있는 경우

```kotlin
@PostMapping("/actions/refresh")
fun refresh(@RequestBody(required = false) req: RefreshRequest?): TokenResponse = ...
```

---

## 5. `@RequestHeader` — HTTP header

```kotlin
@GetMapping("/secret")
fun secret(
    @RequestHeader("Authorization") auth: String,
    @RequestHeader("X-Idempotency-Key", required = false) idemKey: String?,
    @RequestHeader("Accept-Language", defaultValue = "ko") lang: String,
): Any = ...
```

### 5.1 모든 header 받기

```kotlin
@GetMapping("/debug/headers")
fun headers(@RequestHeader headers: Map<String, String>): Map<String, String> = headers
```

---

## 6. `@CookieValue`

```kotlin
@GetMapping("/")
fun home(
    @CookieValue("JSESSIONID", required = false) session: String?,
): String = ...
```

---

## 7. `@AuthenticationPrincipal`

```kotlin
@GetMapping("/me")
fun me(@AuthenticationPrincipal user: User): UserResponse = UserResponse.from(user)
```

→ Spring Security 의 SecurityContext 에서 인증된 user 추출. [[05-security-and-deployment#5 JwtAuthenticationFilter|05 §5]] 의 filter 가 `User` 객체를 principal 로 설정해야 동작.

### 7.1 type 미스매치 시 null

```kotlin
@GetMapping("/me")
fun me(@AuthenticationPrincipal user: User?): UserResponse? {
    if (user == null) throw UnauthorizedException()
    return UserResponse.from(user)
}
```

→ `user: User` non-null 인데 SecurityContext 가 비어있으면 NPE. `User?` 로 안전.

---

## 8. 파일 업로드 — `MultipartFile`

```kotlin
import org.springframework.web.multipart.MultipartFile

@PostMapping("/upload", consumes = ["multipart/form-data"])
fun upload(
    @RequestParam("file") file: MultipartFile,
    @RequestParam("description", required = false) description: String?,
): UploadResponse {
    if (file.isEmpty) throw BusinessException(400, "EMPTY_FILE", "파일 비어있음")

    val bytes = file.bytes
    val originalName = file.originalFilename
    val size = file.size
    val contentType = file.contentType

    // S3 업로드 / DB 저장 등
    return UploadResponse(id = "...", size = size)
}
```

> 큰 파일은 [[../api-design/file-upload-s3]] 의 presigned URL 패턴 권장 (서버 거치지 않음).

### 8.1 multipart limit

```yaml
# application.yml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 50MB
```

---

## 9. 응답 — return type 별 처리

### 9.1 객체 → JSON 자동

```kotlin
@GetMapping("/users/{id}")
fun get(@PathVariable id: Long): UserResponse = ...
// → 200 + JSON body
```

### 9.2 `ResponseEntity<T>` — 명시적 status / header

```kotlin
@PostMapping("/users")
fun create(@Valid @RequestBody req: CreateUserRequest): ResponseEntity<UserResponse> {
    val user = userService.create(req)
    return ResponseEntity
        .created(URI.create("/users/${user.id}"))             // 201 + Location 헤더
        .body(UserResponse.from(user))
}

@DeleteMapping("/users/{id}")
fun delete(@PathVariable id: Long): ResponseEntity<Void> {
    userService.delete(id)
    return ResponseEntity.noContent().build()                  // 204
}
```

### 9.3 `@ResponseStatus` — status 만 명시

```kotlin
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)
fun create(@Valid @RequestBody req: CreateUserRequest): UserResponse = ...

@DeleteMapping("/users/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)
fun delete(@PathVariable id: Long) {
    userService.delete(id)
}
```

→ 단순하면 `@ResponseStatus`, header 도 필요하면 `ResponseEntity`.

### 9.4 binary 응답 — 파일 다운로드

```kotlin
@GetMapping("/files/{id}/download")
fun download(@PathVariable id: Long): ResponseEntity<ByteArray> {
    val file = fileService.findById(id)
    val bytes = s3Service.download(file.s3Key)

    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType(file.contentType))
        .header(HttpHeaders.CONTENT_DISPOSITION, """attachment; filename="${file.name}"""")
        .body(bytes)
}
```

### 9.5 streaming 응답 — 큰 파일 / CSV

```kotlin
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody

@GetMapping("/exports/users.csv")
fun exportCsv(): ResponseEntity<StreamingResponseBody> {
    val body = StreamingResponseBody { output ->
        OutputStreamWriter(output, Charsets.UTF_8).use { writer ->
            writer.write("﻿")                              // UTF-8 BOM (엑셀 한글)
            writer.write("id,email,name\n")
            userRepository.findAll().forEach {
                writer.write("${it.id},${it.email},${it.name}\n")
                writer.flush()
            }
        }
    }
    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
        .header(HttpHeaders.CONTENT_DISPOSITION, """attachment; filename="users.csv"""")
        .body(body)
}
```

### 9.6 redirect

```kotlin
@GetMapping("/old-path")
fun redirect(): ResponseEntity<Void> = ResponseEntity
    .status(HttpStatus.MOVED_PERMANENTLY)
    .header(HttpHeaders.LOCATION, "/new-path")
    .build()
```

---

## 10. status code 결정 — REST 관행

| 상황 | status | 설명 |
| --- | --- | --- |
| 정상 조회 / 갱신 | **200 OK** | |
| 신규 리소스 생성 | **201 Created** | Location 헤더 권장 |
| 처리 시작 (async) | **202 Accepted** | webhook / job |
| 정상 삭제 / body 없음 | **204 No Content** | body X |
| client 입력 잘못 | **400 Bad Request** | 의미 자체 잘못 |
| 인증 안 됨 | **401 Unauthorized** | 토큰 없음 / 무효 |
| 권한 없음 | **403 Forbidden** | 인증은 됐지만 권한 X |
| 리소스 없음 | **404 Not Found** | |
| method 안 됨 | **405 Method Not Allowed** | GET 만 가능한 path 에 POST |
| 충돌 | **409 Conflict** | unique 위반 / 동시 변경 |
| validation 실패 | **422 Unprocessable Entity** | format 은 맞지만 의미 불가 |
| 요청 too many | **429 Too Many Requests** | rate limit |
| 서버 오류 | **500 Internal Server Error** | 캐치올 |
| 외부 의존 실패 | **502 Bad Gateway** | upstream API fail |
| 일시 사용 불가 | **503 Service Unavailable** | maintenance |

→ **422 vs 400**: 본 vault 는 Bean Validation 실패 = 422. parsing 자체 실패 (JSON malformed) = 400.

---

## 11. CORS 패턴

전역 (`SecurityConfig + CorsConfigurationSource`) 외에 endpoint 단:

```kotlin
@CrossOrigin(origins = ["https://shop.example.com"], maxAge = 3600)
@GetMapping("/public/widget")
fun widget(): String = ...
```

> 보통 전역 CORS 사용. 특정 endpoint 만 다른 origin 허용 시 `@CrossOrigin`.

---

## 12. Content negotiation — JSON / XML / YAML

```kotlin
@GetMapping("/users/{id}", produces = ["application/json", "application/xml"])
fun get(@PathVariable id: Long): UserResponse = ...
```

→ client 가 `Accept: application/xml` 보내면 XML 응답 (Jackson XML 의존성 필요). 보통 JSON only.

---

## 13. Versioning 패턴

| 방식 | 예시 |
| --- | --- |
| **URL prefix** | `/api/v1/users` → `/api/v2/users` (권장) |
| Custom header | `Accept: application/vnd.shop.v2+json` |
| Query parameter | `?version=2` |

본 vault 는 **URL prefix** 표준 (`/api/v1/...`).

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserV1Controller { ... }

@RestController
@RequestMapping("/api/v2/users")
class UserV2Controller { ... }
```

---

## 14. 자주 막히는 함정

### 함정 1 — `@PathVariable` 의 type 자동 변환 실패
URL `/users/abc` 에 `@PathVariable id: Long` → 400 (`Failed to convert`). 정상 — type-safe.

### 함정 2 — `@RequestBody` 가 GET 에
HTTP spec 상 GET 의 body 는 정의 안 됨. POST/PUT/PATCH 만.

### 함정 3 — `@AuthenticationPrincipal user: User` null
SecurityConfig 의 filter 가 principal 안 채움. **`User?` 로 받고 null 체크** 또는 SecurityConfig 점검.

### 함정 4 — `Pageable` 의 sort 직접 받기
`?sort=createdAt,desc` 만 지원. 잘못된 컬럼명 = SQL 에러. **화이트리스트 검증**.

### 함정 5 — `MultipartFile` 의 메모리 폭발
default 가 `MultipartFile.getBytes()` → 전체 메모리. **`getInputStream()`** 으로 streaming 또는 presigned upload.

### 함정 6 — `ResponseEntity<Unit>` 가 응답 body 에 `{}` 노출
204 No Content 의도면 `ResponseEntity<Void>` 또는 `.noContent().build()`.

### 함정 7 — `@ResponseStatus(HttpStatus.CREATED)` 가 항상 201
예외 던질 때도 201? **`@ExceptionHandler` 가 우선** — 정상 흐름만 201.

### 함정 8 — `@CrossOrigin` 과 전역 CORS 충돌
`SecurityConfig` 의 CorsConfiguration 과 둘 다 있으면 어느 게 우선인지 모호. **하나만**.

### 함정 9 — `@RequestParam Map<String, String>` 가 path variable 까지 받음?
No — query string 만. path variable 은 `@PathVariable Map<String, String>` (잘 안 씀).

### 함정 10 — Jackson 이 LocalDateTime 직렬화 fail
default `WRITE_DATES_AS_TIMESTAMPS=true` → `[2026,5,27,...]` 형식. **`JavaTimeModule` 등록 + `WRITE_DATES_AS_TIMESTAMPS=false`** ([[02-project-structure#4 Jackson 설정|02 §4]]).

---

## 15. 다음

✅ Controller 패턴 끝.

→ [[08-actuator-monitoring]] — Spring Boot Actuator + healthcheck + Prometheus
