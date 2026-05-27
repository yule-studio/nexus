---
title: "06. Swagger UI / springdoc-openapi 통합"
kind: guide
status: current
created_at: 2026-05-27T11:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, swagger, openapi, springdoc]
---

# 06. Swagger UI / springdoc-openapi 통합

**[[guidelines|↑ Hub]]** ← [[05-security-and-deployment]] · → 다음 [[07-controller-patterns]]

> **springdoc-openapi** = Spring Boot 의 표준 OpenAPI / Swagger 통합 라이브러리.
> 클래스 / 메서드 / DTO 의 어노테이션만 보고 `/swagger-ui/index.html` 에 인터랙티브 API 문서 자동 생성.

---

## 1. 왜 Swagger UI 인가

- **클라이언트 (FE / 외부 파트너) 가 항상 최신 spec 확인**.
- **Postman 없이 브라우저에서 바로 시험** — JWT 인증 한 번 입력 후 모든 endpoint 호출.
- **OpenAPI 3.x 스펙** 표준 — 자동 client SDK 생성 / Postman 가져오기 / Stoplight 등.

### 1.1 FastAPI 와 비교

FastAPI 는 native `/docs` 자동 — Spring 은 별도 라이브러리 (springdoc-openapi). 하지만 한 번 셋업하면 동일하게 자동.

---

## 2. 의존성 추가

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0")
}
```

> springdoc-openapi 2.x = Spring Boot 3 + Jakarta. (1.x 는 Spring Boot 2 + Javax — 호환 X.)

---

## 3. 첫 동작 — 의존성만 추가하면 끝

```bash
./gradlew bootRun
```

- **Swagger UI**: <http://localhost:8080/swagger-ui/index.html>
- **OpenAPI JSON**: <http://localhost:8080/v3/api-docs>
- **OpenAPI YAML**: <http://localhost:8080/v3/api-docs.yaml>

이 시점에 이미 모든 `@RestController` 의 endpoint 가 표시됨. DTO 도 schema 자동 추출.

> **권한 함정**: Spring Security 가 active 이면 swagger-ui 자체가 401. 다음 §4 참고.

---

## 4. SecurityConfig 에서 swagger 경로 허용

```kotlin
// 05-security-and-deployment.md 의 SecurityConfig 의 authorizeHttpRequests 안에 추가
authorizeHttpRequests {
    authorize("/api/v1/auth/signup", permitAll)
    authorize("/api/v1/auth/login", permitAll)

    // ↓ swagger 정적 자원 / spec 허용
    authorize("/swagger-ui/**", permitAll)
    authorize("/swagger-ui.html", permitAll)
    authorize("/v3/api-docs/**", permitAll)
    authorize("/v3/api-docs.yaml", permitAll)
    authorize("/swagger-resources/**", permitAll)
    authorize("/webjars/**", permitAll)

    authorize("/actuator/health", permitAll)
    authorize("/actuator/info", permitAll)
    authorize(anyRequest, authenticated)
}
```

운영에선 swagger 노출 안 하려면 profile 별 분기 또는 `springdoc.api-docs.enabled=false`.

---

## 5. application.yml — springdoc 옵션

```yaml
# src/main/resources/application.yml
springdoc:
  api-docs:
    enabled: true                          # OpenAPI JSON 생성
    path: /v3/api-docs                     # default
  swagger-ui:
    enabled: true                          # Swagger UI 활성
    path: /swagger-ui.html                 # /swagger-ui/index.html 으로 redirect
    operations-sorter: method               # GET/POST/... 순서로 정렬
    tags-sorter: alpha                      # 알파벳 순
    display-request-duration: true          # 요청 시간 표시
    try-it-out-enabled: true
  packages-to-scan: com.example.memo         # scan 범위 (default 전체)
  default-produces-media-type: application/json
  default-consumes-media-type: application/json
```

운영에선 비활성:

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

---

## 6. OpenAPI 메타데이터 — `OpenAPIDefinition`

기본 title / version / description / contact 설정:

```kotlin
// src/main/kotlin/com/example/memo/config/OpenApiConfig.kt
package com.example.memo.config

import io.swagger.v3.oas.models.Components
import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Contact
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import io.swagger.v3.oas.models.security.SecurityRequirement
import io.swagger.v3.oas.models.security.SecurityScheme
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {

    @Bean
    fun openAPI(): OpenAPI = OpenAPI()
        .info(
            Info()
                .title("Memo API")
                .description("간단한 메모 CRUD + 회원가입 / JWT 인증")
                .version("v1.0.0")
                .contact(Contact().name("yuchan").email("yuchan@example.com"))
                .license(License().name("Apache 2.0").url("https://www.apache.org/licenses/LICENSE-2.0")),
        )
        // 모든 endpoint 에 JWT Bearer 인증 적용
        .addSecurityItem(SecurityRequirement().addList("BearerAuth"))
        .components(
            Components().addSecuritySchemes(
                "BearerAuth",
                SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
                    .`in`(SecurityScheme.In.HEADER)
                    .name("Authorization"),
            ),
        )
}
```

> Swagger UI 우측 상단에 `Authorize` 버튼 자동 생성. 클릭 → JWT 입력 → 모든 endpoint 호출 시 자동 헤더 추가.
>
> **Kotlin 한정**: `in` 은 reserved keyword → 백틱 escape `` `in` ``.

---

## 7. Controller / DTO 어노테이션 — 더 풍부한 문서

```kotlin
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.Parameter
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse as SwaggerResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import io.swagger.v3.oas.annotations.tags.Tag


@Tag(name = "memo", description = "메모 CRUD")
@RestController
@RequestMapping("/api/v1/memos")
class MemoController(
    private val memoService: MemoService,
) {

    @Operation(
        summary = "내 메모 목록",
        description = "현재 로그인된 사용자의 메모를 페이지 단위로 조회. 최신 순.",
    )
    @ApiResponses(
        value = [
            SwaggerResponse(
                responseCode = "200", description = "조회 성공",
                content = [Content(mediaType = "application/json", schema = Schema(implementation = ApiResponse::class))],
            ),
            SwaggerResponse(responseCode = "401", description = "인증 필요"),
        ],
    )
    @GetMapping
    fun list(
        @AuthenticationPrincipal user: User,
        @Parameter(description = "0-based page index", example = "0")
        @RequestParam(defaultValue = "0") page: Int,
        @Parameter(description = "한 페이지의 항목 수", example = "20")
        @RequestParam(defaultValue = "20") size: Int,
    ): ApiResponse<List<MemoResponse>> {
        val memos = memoService.listByOwner(user.id!!, page, size)
        return ApiResponse.ok(memos.map(MemoResponse::from), "조회 성공")
    }

    @Operation(summary = "메모 생성")
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            required = true,
            content = [Content(schema = Schema(implementation = CreateMemoRequest::class))],
        )
        @Valid @org.springframework.web.bind.annotation.RequestBody req: CreateMemoRequest,
        @AuthenticationPrincipal user: User,
    ): ApiResponse<MemoResponse> {
        val memo = memoService.create(user, req.title, req.content)
        return ApiResponse.ok(MemoResponse.from(memo), "생성 완료")
    }
}
```

> **함정**: `@RequestBody` 가 두 종류 있음 — Swagger 어노테이션 (`io.swagger.v3.oas.annotations.parameters.RequestBody`) vs Spring 어노테이션 (`org.springframework.web.bind.annotation.RequestBody`). 같이 쓸 때 FQCN 또는 import alias.

### 7.1 DTO 에 Schema 메타

```kotlin
import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.Size


@Schema(description = "메모 생성 요청")
data class CreateMemoRequest(
    @field:Schema(description = "제목", example = "첫 메모", required = true)
    @field:Size(min = 1, max = 200)
    val title: String,

    @field:Schema(description = "본문 (선택)", example = "안녕하세요")
    @field:Size(max = 10_000)
    val content: String = "",
)


@Schema(description = "메모 응답")
data class MemoResponse(
    @field:Schema(description = "메모 ID", example = "1")
    val id: Long,

    @field:Schema(description = "제목")
    val title: String,
    val content: String,
    val createdAt: Instant,
    val updatedAt: Instant,
)
```

→ Swagger UI 의 schema 탭 / `Example Value` 에 자동 반영.

---

## 8. Group / Tag — 큰 프로젝트 정리

```kotlin
@Bean
fun authGroup(): GroupedOpenApi = GroupedOpenApi.builder()
    .group("auth")
    .pathsToMatch("/api/v1/auth/**")
    .build()

@Bean
fun memoGroup(): GroupedOpenApi = GroupedOpenApi.builder()
    .group("memo")
    .pathsToMatch("/api/v1/memos/**")
    .build()
```

→ Swagger UI 우측 상단 dropdown 에서 group 선택. 도메인 많은 프로젝트에서 navigation 편함.

---

## 9. 보안 — 운영 시 비활성

운영에서 swagger 노출 = API 구조 공개 + 공격 surface. **`prod` profile 에서 비활성** 또는 admin 만 접근:

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
```

또는 SecurityConfig 에서 admin 만:

```kotlin
authorize("/swagger-ui/**", hasRole("ADMIN"))
authorize("/v3/api-docs/**", hasRole("ADMIN"))
```

---

## 10. 자주 막히는 함정

### 함정 1 — Swagger UI 가 401
SecurityConfig 의 `authorizeHttpRequests` 에 `/swagger-ui/**` `/v3/api-docs/**` 누락. **permitAll 추가**.

### 함정 2 — springdoc 2.x vs 1.x
Spring Boot 3 → springdoc 2.x. Spring Boot 2 → springdoc 1.x. **버전 매칭**.

### 함정 3 — `@io.swagger.v3.oas.annotations.parameters.RequestBody` vs `@org.springframework.web.bind.annotation.RequestBody`
이름 같지만 다른 어노테이션. **둘 다 명시** 또는 FQCN.

### 함정 4 — Kotlin 의 nullable 가 schema 에 안 반영
`val title: String?` 인데 schema 가 required true 로 노출. 명시:
```kotlin
@field:Schema(required = false, nullable = true)
val title: String? = null
```

### 함정 5 — `BearerAuth` SecurityRequirement 등록 안 함
Swagger UI 의 `Authorize` 버튼이 없음. `OpenApi` bean 의 `addSecurityItem` + `addSecuritySchemes` 둘 다 필요.

### 함정 6 — `springdoc.packages-to-scan` 미설정
default 는 전체 scan — 큰 프로젝트면 느림. **명시 권장**.

### 함정 7 — `@Tag` 이름이 다른 controller 에 중복
같은 이름 = Swagger UI 에서 한 group 으로 묶임. 의도면 OK, 아니면 다르게.

### 함정 8 — 운영 swagger 노출
**`enabled: false`** 또는 admin only.

---

## 11. 추가 — Postman 가져오기 / SDK 생성

OpenAPI JSON 다운로드:
```bash
curl http://localhost:8080/v3/api-docs > openapi.json
```

- **Postman**: `Import` → `openapi.json` → Collection 자동 생성.
- **TypeScript SDK**: `openapi-typescript` 또는 `openapi-generator-cli`.
- **Java client**: `openapi-generator-cli generate -i openapi.json -g java -o ./client`.

---

## 12. 다음

✅ Swagger UI 통합 끝.

→ [[07-controller-patterns]] — `@PathVariable` / `@RequestParam` / `@RequestBody` / 다양한 응답 형식 패턴
