---
title: "05. Spring Security + JWT + Docker 배포"
kind: guide
status: current
created_at: 2026-05-27T10:50:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, security, jwt, deployment, docker]
---

# 05. Spring Security + JWT + Docker 배포

**[[guidelines|↑ Hub]]** ← [[04-controller-and-validation]]

> Spring Security 6 + JWT (jjwt 0.12) + Argon2 + Docker 배포.

---

## 1. Spring Security 6 — 기본 흐름

```
HTTP request
   ↓
[Filter Chain]
   ├─ CorsFilter
   ├─ JwtAuthenticationFilter      ← 우리가 추가 (Authorization 헤더에서 JWT 파싱 + SecurityContext 설정)
   ├─ UsernamePasswordAuthenticationFilter
   └─ ... (Spring Security 내장 filter)
   ↓
[DispatcherServlet → @RestController]
   ↑
@AuthenticationPrincipal user: User    ← SecurityContext 에서 추출
```

→ **JWT 검증은 Filter 단**. Controller 는 인증된 user 만 받음.

---

## 2. 의존성 추가

```kotlin
// build.gradle.kts
dependencies {
    // 이미 있음
    implementation("org.springframework.boot:spring-boot-starter-security")

    // JWT 추가
    implementation("io.jsonwebtoken:jjwt-api:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")

    // 비밀번호 hash — Argon2
    implementation("org.springframework.security:spring-security-crypto")
}
```

---

## 3. PasswordEncoder — Argon2

```kotlin
// src/main/kotlin/com/example/memo/config/PasswordConfig.kt
package com.example.memo.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.crypto.argon2.Argon2PasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder

@Configuration
class PasswordConfig {

    @Bean
    fun passwordEncoder(): PasswordEncoder = Argon2PasswordEncoder(
        16,                        // salt length
        32,                        // hash length
        2,                         // parallelism
        65536,                     // memory (64 MB)
        3,                         // iterations
    )
}
```

> **Argon2id** (default — argon2-cffi 의 Argon2id) > bcrypt > pbkdf2. 단방향. salt 자동.
>
> Spring Security 의 `BCryptPasswordEncoder()` 가 더 간단하지만 본 가이드는 Argon2 권장.

사용:

```kotlin
@Service
class SignupService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
) {
    @Transactional
    fun signup(email: String, plainPassword: String, name: String): User {
        if (userRepository.existsByEmail(email)) {
            throw ConflictException("이미 가입된 이메일입니다.")
        }
        val user = User(
            email = email,
            passwordHash = passwordEncoder.encode(plainPassword),
            name = name,
        )
        return userRepository.save(user)
    }
}
```

```kotlin
// login 시 검증
val matches = passwordEncoder.matches(plainPassword, user.passwordHash)
if (!matches) throw UnauthorizedException("비밀번호가 올바르지 않습니다.")
```

---

## 4. JwtTokenProvider

```kotlin
// src/main/kotlin/com/example/memo/security/JwtTokenProvider.kt
package com.example.memo.security

import com.example.memo.config.AppProperties
import io.jsonwebtoken.Claims
import io.jsonwebtoken.ExpiredJwtException
import io.jsonwebtoken.Jwts
import io.jsonwebtoken.JwtException
import io.jsonwebtoken.security.Keys
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component
import java.time.Duration
import java.time.Instant
import java.util.Base64
import java.util.Date
import java.util.UUID
import javax.crypto.SecretKey

@Component
class JwtTokenProvider(
    appProperties: AppProperties,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    private val key: SecretKey
    private val accessTtl: Duration = Duration.ofMinutes(appProperties.jwt.accessTokenExpirationMinutes)
    private val refreshTtl: Duration = Duration.ofDays(appProperties.jwt.refreshTokenExpirationDays)
    private val issuer: String = appProperties.jwt.issuer

    init {
        val keyBytes = appProperties.jwt.secret.toByteArray(Charsets.UTF_8)
        require(keyBytes.size >= 32) {
            "JWT secret 은 32 bytes 이상이어야 합니다. (현재 ${keyBytes.size})"
        }
        key = Keys.hmacShaKeyFor(keyBytes)
    }

    fun generateAccessToken(userId: Long, email: String, role: String): String {
        val now = Instant.now()
        return Jwts.builder()
            .issuer(issuer)
            .subject(email)
            .id(UUID.randomUUID().toString())
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(accessTtl)))
            .claim("token_type", "ACCESS")
            .claim("user_id", userId)
            .claim("role", role)
            .signWith(key, Jwts.SIG.HS256)
            .compact()
    }

    fun generateRefreshToken(userId: Long): String {
        val now = Instant.now()
        return Jwts.builder()
            .issuer(issuer)
            .subject(userId.toString())
            .id(UUID.randomUUID().toString())
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(refreshTtl)))
            .claim("token_type", "REFRESH")
            .signWith(key, Jwts.SIG.HS256)
            .compact()
    }

    fun parseClaims(token: String): Claims =
        Jwts.parser()
            .verifyWith(key)
            .requireIssuer(issuer)
            .clockSkewSeconds(30)
            .build()
            .parseSignedClaims(token)
            .payload

    fun validateToken(token: String): Boolean {
        if (token.isBlank()) return false
        return try {
            parseClaims(token); true
        } catch (e: ExpiredJwtException) {
            log.debug("expired token"); false
        } catch (e: JwtException) {
            log.warn("invalid JWT: {}", e.message); false
        } catch (e: IllegalArgumentException) {
            log.warn("empty JWT"); false
        }
    }

    fun isAccess(token: String): Boolean = runCatching {
        parseClaims(token).get("token_type", String::class.java) == "ACCESS"
    }.getOrDefault(false)

    fun getEmail(token: String): String = parseClaims(token).subject
    fun getUserId(token: String): Long = parseClaims(token).get("user_id", java.lang.Long::class.java).toLong()
    fun getRole(token: String): String = parseClaims(token).get("role", String::class.java)
}
```

---

## 5. JwtAuthenticationFilter

```kotlin
// src/main/kotlin/com/example/memo/security/JwtAuthenticationFilter.kt
package com.example.memo.security

import com.example.memo.user.User
import com.example.memo.user.UserRepository
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.LoggerFactory
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

@Component
class JwtAuthenticationFilter(
    private val jwtTokenProvider: JwtTokenProvider,
    private val userRepository: UserRepository,
) : OncePerRequestFilter() {

    private val log = LoggerFactory.getLogger(javaClass)

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        val token = resolveToken(request)

        if (token != null
            && jwtTokenProvider.validateToken(token)
            && jwtTokenProvider.isAccess(token)
        ) {
            val userId = jwtTokenProvider.getUserId(token)
            val user = userRepository.findById(userId).orElse(null)
            if (user != null && user.status == com.example.memo.user.UserStatus.ACTIVE) {
                val authorities = listOf(SimpleGrantedAuthority("ROLE_USER"))
                val auth = UsernamePasswordAuthenticationToken(user, null, authorities)
                SecurityContextHolder.getContext().authentication = auth
            }
        }
        filterChain.doFilter(request, response)
    }

    private fun resolveToken(request: HttpServletRequest): String? {
        val bearer = request.getHeader("Authorization") ?: return null
        return if (bearer.startsWith("Bearer ")) bearer.removePrefix("Bearer ") else null
    }
}
```

> **`OncePerRequestFilter`** = 한 요청에 한 번만 실행 (forward / include 시 중복 방지).
> **`SecurityContextHolder.getContext().authentication = auth`** → `@AuthenticationPrincipal user: User` 가 user 받음.

---

## 6. SecurityConfig

```kotlin
// src/main/kotlin/com/example/memo/config/SecurityConfig.kt
package com.example.memo.config

import com.example.memo.common.ApiResponse
import com.example.memo.security.JwtAuthenticationFilter
import com.fasterxml.jackson.databind.ObjectMapper
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer
import org.springframework.security.config.annotation.web.invoke
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
import org.springframework.web.cors.UrlBasedCorsConfigurationSource

@Configuration
@EnableMethodSecurity
class SecurityConfig(
    private val jwtAuthenticationFilter: JwtAuthenticationFilter,
    private val corsSource: UrlBasedCorsConfigurationSource,
    private val objectMapper: ObjectMapper,
) {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http {
            cors { configurationSource = corsSource }
            csrf { disable() }
            httpBasic { disable() }
            formLogin { disable() }
            sessionManagement {
                sessionCreationPolicy = SessionCreationPolicy.STATELESS
            }

            authorizeHttpRequests {
                authorize("/api/v1/auth/signup", permitAll)
                authorize("/api/v1/auth/login", permitAll)
                authorize("/actuator/health", permitAll)
                authorize("/actuator/info", permitAll)
                authorize(anyRequest, authenticated)
            }

            exceptionHandling {
                authenticationEntryPoint { _, response, _ ->
                    val body = ApiResponse.fail("UNAUTHORIZED", "인증이 필요합니다.")
                    response.status = HttpStatus.UNAUTHORIZED.value()
                    response.contentType = MediaType.APPLICATION_JSON_VALUE
                    response.characterEncoding = "UTF-8"
                    objectMapper.writeValue(response.outputStream, body)
                }
                accessDeniedHandler { _, response, _ ->
                    val body = ApiResponse.fail("FORBIDDEN", "권한이 없습니다.")
                    response.status = HttpStatus.FORBIDDEN.value()
                    response.contentType = MediaType.APPLICATION_JSON_VALUE
                    response.characterEncoding = "UTF-8"
                    objectMapper.writeValue(response.outputStream, body)
                }
            }

            addFilterBefore<UsernamePasswordAuthenticationFilter>(jwtAuthenticationFilter)
        }
        return http.build()
    }
}
```

> **`http { ... }` Kotlin DSL** 의 import 가 핵심 — `org.springframework.security.config.annotation.web.invoke`.
> **`authenticationEntryPoint`** = 미인증 시 401 envelope.
> **`accessDeniedHandler`** = 권한 부족 시 403 envelope.

---

## 7. Auth Controller — signup / login

```kotlin
// src/main/kotlin/com/example/memo/auth/AuthController.kt
package com.example.memo.auth

import com.example.memo.common.ApiResponse
import com.example.memo.common.UnauthorizedException
import com.example.memo.security.JwtTokenProvider
import com.example.memo.user.UserRepository
import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Size
import org.springframework.http.HttpStatus
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.transaction.annotation.Transactional
import org.springframework.web.bind.annotation.*

data class SignupRequest(
    @field:Email val email: String,
    @field:Size(min = 8, max = 64) val password: String,
    @field:Size(min = 1, max = 100) val name: String,
)

data class LoginRequest(
    @field:Email val email: String,
    @field:NotBlank val password: String,
)

data class TokenResponse(
    val accessToken: String,
    val refreshToken: String,
    val tokenType: String = "Bearer",
    val expiresIn: Long = 900,
)

@RestController
@RequestMapping("/api/v1/auth")
class AuthController(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val jwtTokenProvider: JwtTokenProvider,
) {

    @PostMapping("/signup")
    @ResponseStatus(HttpStatus.CREATED)
    @Transactional
    fun signup(@Valid @RequestBody req: SignupRequest): ApiResponse<Map<String, Any>> {
        if (userRepository.existsByEmail(req.email)) {
            throw com.example.memo.common.ConflictException("이미 가입된 이메일입니다.")
        }
        val user = com.example.memo.user.User(
            email = req.email,
            passwordHash = passwordEncoder.encode(req.password),
            name = req.name,
        )
        userRepository.save(user)
        return ApiResponse.ok(
            mapOf("id" to user.id!!, "email" to user.email),
            "회원가입 성공",
        )
    }

    @PostMapping("/login")
    fun login(@Valid @RequestBody req: LoginRequest): ApiResponse<TokenResponse> {
        val user = userRepository.findByEmail(req.email)
            ?: throw UnauthorizedException("이메일 또는 비밀번호가 올바르지 않습니다.")

        if (!passwordEncoder.matches(req.password, user.passwordHash)) {
            throw UnauthorizedException("이메일 또는 비밀번호가 올바르지 않습니다.")
        }

        val access = jwtTokenProvider.generateAccessToken(user.id!!, user.email, "ROLE_USER")
        val refresh = jwtTokenProvider.generateRefreshToken(user.id!!)
        return ApiResponse.ok(TokenResponse(access, refresh), "로그인 성공")
    }
}
```

---

## 8. 실습 — 전체 흐름

```bash
./gradlew bootRun

# 1. signup
curl -X POST http://localhost:8080/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}'
# 201 Created + {"success":true,"data":{"id":1,"email":"..."},"message":"회원가입 성공"}

# 2. login
TOKEN=$(curl -sX POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}' \
  | jq -r '.data.accessToken')

# 3. 메모 생성
curl -X POST http://localhost:8080/api/v1/memos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕하세요"}'
# 201 + {"success":true,"data":{...}}

# 4. 목록
curl http://localhost:8080/api/v1/memos \
  -H "Authorization: Bearer $TOKEN"

# 5. 토큰 없이 호출 → 401
curl http://localhost:8080/api/v1/memos
# 401 + {"success":false,"error":{"code":"UNAUTHORIZED",...}}
```

---

## 9. 테스트 — `@SpringBootTest` + `WebTestClient`

```kotlin
// src/test/kotlin/com/example/memo/MemoIntegrationTest.kt
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class MemoIntegrationTest @Autowired constructor(
    val mockMvc: MockMvc,
    val objectMapper: ObjectMapper,
    val userRepository: UserRepository,
    val passwordEncoder: PasswordEncoder,
    val jwtTokenProvider: JwtTokenProvider,
) {

    @BeforeEach
    fun setup() {
        userRepository.deleteAll()
    }

    @Test
    fun `signup + login + create memo + list`() {
        // signup
        mockMvc.perform(
            post("/api/v1/auth/signup")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"email":"t@x.com","password":"P@ssword1","name":"t"}""")
        ).andExpect(status().isCreated)

        // login → token
        val loginRes = mockMvc.perform(
            post("/api/v1/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"email":"t@x.com","password":"P@ssword1"}""")
        ).andExpect(status().isOk).andReturn().response.contentAsString

        val token = objectMapper.readTree(loginRes).at("/data/accessToken").asText()

        // 메모 생성
        mockMvc.perform(
            post("/api/v1/memos")
                .header("Authorization", "Bearer $token")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"title":"t1","content":"c1"}""")
        ).andExpect(status().isCreated)

        // 목록
        mockMvc.perform(
            get("/api/v1/memos")
                .header("Authorization", "Bearer $token")
        ).andExpect(status().isOk)
         .andExpect(jsonPath("$.data[0].title").value("t1"))
    }
}
```

---

## 10. Docker 배포

### 10.1 Dockerfile

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-jammy

WORKDIR /app

COPY build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

빌드 + 실행:

```bash
./gradlew bootJar
docker build -t memo-api:latest .
docker run -d -p 8080:8080 \
  -e DB_HOST=host.docker.internal \
  -e JWT_SECRET=$(openssl rand -hex 32) \
  memo-api:latest
```

### 10.2 docker-compose

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: memo
      POSTGRES_PASSWORD: memo
      POSTGRES_DB: memo
    ports: ["5432:5432"]
    volumes: ["dbdata:/var/lib/postgresql/data"]

  app:
    build: .
    depends_on: [db]
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_HOST: db
      DB_USER: memo
      DB_PASSWORD: memo
      DB_NAME: memo
      JWT_SECRET: ${JWT_SECRET:-32bytesplease32bytesplease3232!!}
    ports: ["8080:8080"]

volumes:
  dbdata:
```

```bash
./gradlew bootJar
docker compose up --build
```

### 10.3 Multi-stage build (이미지 크기 ↓)

```dockerfile
# 1. Build stage
FROM gradle:8.5-jdk21 AS build
WORKDIR /src
COPY . .
RUN gradle bootJar --no-daemon

# 2. Runtime stage
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY --from=build /src/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

## 11. 운영 체크리스트

- [ ] `SPRING_PROFILES_ACTIVE=prod` (default `local` 금지)
- [ ] `JWT_SECRET` 가 vault / Secrets Manager (env 직접 X)
- [ ] HTTPS (Nginx / ALB TLS termination)
- [ ] `spring.jpa.hibernate.ddl-auto = validate`
- [ ] `spring.jpa.open-in-view = false`
- [ ] Flyway upgrade 가 컨테이너 startup 자동 (기본 동작)
- [ ] PostgreSQL `max_connections` ≥ replicas × pool_size
- [ ] log → stdout (Docker / k8s 가 수집)
- [ ] healthcheck `/actuator/health` 사용
- [ ] CORS `allowed-origins` 명시 (`*` X)
- [ ] `@PreAuthorize` 필요한 endpoint (`@EnableMethodSecurity`)
- [ ] 비밀번호 Argon2 또는 bcrypt cost 12+
- [ ] JWT secret 32 bytes 이상
- [ ] Refresh token rotation (옵션 — [[../api-design/api-design]] 의 login-jwt 레시피)

---

## 12. 자주 막히는 함정

### 함정 1 — `org.springframework.security.config.annotation.web.invoke` import 누락
`http { ... }` 안 됨. import 확인.

### 함정 2 — `JwtAuthenticationFilter` 가 `@Component` 이면서 SecurityConfig 에 주입
순환 의존성 가능. 의존성 트리 확인 — UserRepository → JwtTokenProvider → AppProperties 가 순환 X 여야.

### 함정 3 — JWT secret 가 너무 짧음
`require(keyBytes.size >= 32)` 가 startup 시 fail. 32 bytes (256 bits) 이상.

### 함정 4 — Argon2 cost 너무 큼
운영 트래픽에 1초+ 걸림. 본 가이드의 (3, 65536, 2) = ~100ms. 환경에 맞게 조정.

### 함정 5 — `authorizeHttpRequests` 안 패턴 순서
**구체적 → 일반** 순. `anyRequest` 가 가장 마지막.

### 함정 6 — `SecurityContextHolder.getContext().authentication = auth` 대신 `setAuthentication` 호출
둘 다 OK. Kotlin property syntax.

### 함정 7 — `@AuthenticationPrincipal user: User` 가 null
JwtAuthenticationFilter 가 token validate 실패 (logout 만료 등) → SecurityConfig 의 `authenticationEntryPoint` 가 401 응답 (Controller 도달 X). 정상.

### 함정 8 — `passwordEncoder.encode(...)` 매번 다른 hash
salt 가 매번 랜덤이라 정상. `matches(...)` 가 검증 담당.

### 함정 9 — Refresh token 발급 후 server 가 기록 안 함
black list / DB 저장 안 하면 logout 강제 불가. 본 가이드는 단순 발급만 — 실전은 [[../api-design/api-design]] 의 login-jwt 참고.

### 함정 10 — Docker container 시간대
JVM default UTC. 한국 시간 필요 시 `ENV TZ=Asia/Seoul` (단 DB 시간과 동기 주의 — 가능하면 모두 UTC 권장).

---

## 13. 다음 학습 로드맵

| 단계 | 학습 | 노트 |
| --- | --- | --- |
| 1 | 본 가이드 직접 따라 만들기 | (이 5 파일) |
| 2 | 응답 envelope / Security 표준 깊이 | [[../common/response-envelope]] · [[../common/security-config]] |
| 3 | 실전 레시피 (rate limit / file upload / webhook / 2FA / RBAC ...) | [[../api-design/api-design]] |
| 4 | JPA 깊이 (N+1 / 트랜잭션 / OSIV) | [[../database/jpa]] |
| 5 | MyBatis 함께 사용 | [[../database/mybatis]] / [[../database/jpa-mybatis-coexist]] |
| 6 | Kotlin Coroutines + Spring (WebFlux / Reactive) | (vault 외부 — 추후) |

---

## 14. 끝 — 전체 가이드 정리

✅ 5 단계 완주!

```
01. 설치 + Hello World          ← JDK / Initializr / 첫 endpoint
02. 프로젝트 구조 + 설정          ← 패키지 / yaml / Config
03. Domain + JPA + Flyway        ← Entity / Repository / 마이그레이션
04. Controller + Validation      ← REST / DTO / 예외
05. Security + 배포              ← JWT / Argon2 / Docker  ← 지금
```

이제 [[../api-design/api-design]] 으로 가서 실전 레시피 학습.

---

## 15. 참고 / cross-link

- [[guidelines|↑ Guidelines hub]]
- [[../kotlin-spring|↑ Kotlin Spring]]
- [[../../../python-flask/getting-started]] — Flask 변형
- [[../../../python-fastapi/getting-started]] — FastAPI 변형
- [[../../../python-django/getting-started]] — Django 변형
- Spring Boot 공식: <https://spring.io/projects/spring-boot>
- Spring Security 공식: <https://docs.spring.io/spring-security/reference/>
- jjwt: <https://github.com/jwtk/jjwt>
- Kotlin + Spring: <https://docs.spring.io/spring-framework/reference/languages/kotlin.html>
