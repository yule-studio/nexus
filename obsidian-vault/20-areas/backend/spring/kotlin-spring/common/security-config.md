---
title: "SecurityConfig + JWT Filter + CORS — 표준 세팅 (Kotlin)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T10:30:00+09:00
tags:
  - backend
  - kotlin-spring
  - common
  - security
  - jwt
  - cors
  - production
---

# SecurityConfig + JWT Filter + CORS — 표준 세팅 (Kotlin)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin idiom (DSL / data class / constructor injection / extension) |

**[[../kotlin-spring|↑ Kotlin Spring]]** · **[[../../java-spring/common/security-config|↗ Java 원본]]**

> 본 문서가 **Spring Security 6 + Kotlin 표준 세팅의 본거지**. 모든 레시피의 `SecurityConfig` 는 여기 패턴.
> [[response-envelope]] 의 `CommonResponse / ResponseCode` 를 401/403 응답에 사용.

---

## 1. 컴포넌트 한눈에

```
HttpSecurity (Spring Security 6 Kotlin DSL — invoke operator)
   │
   ├─ cors                       ← CorsConfigurationSource (yaml 의 cors 속성)
   ├─ csrf { disable() }         ← stateless API
   ├─ httpBasic { disable() }
   ├─ sessionManagement { SessionCreationPolicy.STATELESS }
   ├─ authorizeHttpRequests
   │     ├ public:    /api/auth/**, /swagger-ui/**, /actuator/health, ...
   │     ├ MASTER:    /admin/key/**
   │     ├ ADMIN:     /admin/**
   │     └ 그 외:     authenticated()
   ├─ exceptionHandling
   │     ├ authenticationEntryPoint = JwtAuthenticationEntryPoint  (401 응답)
   │     └ accessDeniedHandler     = JwtAccessDeniedHandler        (403 응답)
   ├─ addFilterBefore: ApiLogFilter ─ JwtAuthenticationFilter ─ UsernamePasswordAuthenticationFilter
   │   (요청 들어오는 순서: ApiLog → Jwt → Username)
   └─ build()

빈:
  PasswordEncoder = BCryptPasswordEncoder
  CorsConfigurationSource (CorsProperties 주입)
```

---

## 2. `SecurityConfig`

```kotlin
// src/main/kotlin/com/example/shop/common/config/SecurityConfig.kt
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)        // @PreAuthorize 활성
@EnableConfigurationProperties(CorsProperties::class)
class SecurityConfig(
    private val corsProperties: CorsProperties,
    private val jwtAuthenticationEntryPoint: JwtAuthenticationEntryPoint,
    private val jwtAuthenticationFilter: JwtAuthenticationFilter,
    private val apiLogFilter: ApiLogFilter,
    private val jwtAccessDeniedHandler: JwtAccessDeniedHandler,  // 403 — 신규
) {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain = http {
        cors { configurationSource = corsConfigurationSource() }
        csrf { disable() }
        httpBasic { disable() }
        formLogin { disable() }
        sessionManagement { sessionCreationPolicy = SessionCreationPolicy.STATELESS }

        exceptionHandling {
            authenticationEntryPoint = jwtAuthenticationEntryPoint        // 401 — 미인증
            accessDeniedHandler = jwtAccessDeniedHandler                  // 403 — 인가 실패
        }

        authorizeHttpRequests {
            // ----- 정적 / 공개 -----
            authorize(
                anyOf(
                    "/.well-known/**",
                    "/swagger-ui.html", "/swagger-ui/**",
                    "/v3/api-docs/**", "/swagger-resources/**", "/webjars/**",
                    "/actuator/health", "/actuator/info", "/actuator/prometheus",
                    "/favicon.ico",
                ),
                permitAll,
            )
            // ----- 인증 / 사용자 / 약관 -----
            authorize(
                anyOf(
                    "/api/v1/auth/signup", "/api/v1/auth/signup/**",
                    "/api/v1/auth/login",  "/api/v1/auth/login/**",
                    "/api/v1/auth/token/refresh",
                    "/api/v1/auth/password/**",
                    "/api/v1/auth/duplicate/**",
                    "/api/v1/auth/verify/**",          // 전화번호 / 이메일 인증
                ),
                permitAll,
            )
            // ----- 메시지 / 외부 webhook -----
            authorize(
                anyOf(
                    "/api/v1/messages/webhooks/**",
                    "/api/v1/payments/webhooks/**",
                ),
                permitAll,
            )
            // ----- WebSocket -----
            authorize("/api/v1/ws/**", permitAll)
            // ----- 관리자 -----
            authorize("/admin/key/**", hasRole("MASTER"))
            authorize("/admin/**", hasAnyRole("ADMIN", "MASTER"))
            // ----- 그 외 -----
            authorize(anyRequest, authenticated)
        }

        // 필터 순서: 가장 먼저 들어오는 요청부터 — ApiLog → Jwt → (UsernamePassword)
        addFilterBefore<UsernamePasswordAuthenticationFilter>(jwtAuthenticationFilter)
        addFilterBefore<JwtAuthenticationFilter>(apiLogFilter)
    }.let { http.build() }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()

    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val config = CorsConfiguration().apply {
            allowCredentials = false                       // JWT + Authorization header → credentials false 권장
            allowedOrigins = corsProperties.allowedOrigins.splitOrEmpty()
            allowedMethods = corsProperties.allowedMethods.splitOrEmpty()
            allowedHeaders = corsProperties.allowedHeaders.splitOrEmpty()
            corsProperties.maxAge?.let { maxAge = it }
        }
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/**", config)
        }
    }
}

/** CSV 문자열을 trim + filter 한 List 로. null / blank 안전. */
private fun String?.splitOrEmpty(): List<String> =
    this?.split(",")?.map(String::trim)?.filter { it.isNotEmpty() } ?: emptyList()

private inline fun anyOf(vararg patterns: String): Array<out String> = patterns
```

> **Kotlin idiom 포인트**:
> - `org.springframework.security.config.annotation.web.invoke` 의 **Kotlin DSL** 사용 → Java 의 `http.cors(c -> c.configurationSource(...))` 식 람다 chain 이 자연스러운 builder 로.
> - `apply { }` scope function → 객체 초기화를 한 블록 안에서 mutable property 로 (CorsConfiguration / UrlBasedCorsConfigurationSource).
> - `String?.splitOrEmpty()` extension → Java 의 static helper 가 receiver type 으로.
> - Lombok `@RequiredArgsConstructor` 불필요 — primary constructor 의 `val` 이 자동 주입.
> - `addFilterBefore<UsernamePasswordAuthenticationFilter>(jwtAuthenticationFilter)` — reified type parameter.

### 2.1 개선 사항 (원본 대비)

| 원본 | 개선 |
| --- | --- |
| `accessDeniedHandler` 없음 (403 시 default) | `JwtAccessDeniedHandler` 추가 → 일관된 envelope |
| `formLogin` 비활성 명시 안 함 | `formLogin { disable() }` 추가 |
| `corsProperties.getAllowedOrigins().split(",")` 직접 | `String?.splitOrEmpty()` extension — null / blank 방어 + trim |
| URL list 가 길고 흩어짐 | 그룹별 주석 정렬 |
| `/api/*/user/login/**` 같은 wildcard | path versioning (`/api/v1/auth/login`) 으로 정리 |
| Java builder chain | Kotlin DSL — Spring Security 6 권장 형태 |

---

## 3. `JwtAuthenticationFilter`

```kotlin
// src/main/kotlin/com/example/shop/common/config/jwt/JwtAuthenticationFilter.kt
@Component
class JwtAuthenticationFilter(
    private val jwtTokenProvider: JwtTokenProvider,
    private val userQueryService: UserQueryService,        // emailHash + soft-delete 인지
) : OncePerRequestFilter() {

    private val log = LoggerFactory.getLogger(javaClass)

    companion object {
        private val SWAGGER_PATHS = listOf(
            "/swagger-ui/**", "/swagger-ui.html",
            "/v3/api-docs/**", "/swagger-resources/**", "/webjars/**",
        )
        private val PATH_MATCHER: PathMatcher = AntPathMatcher()
    }

    override fun shouldNotFilter(req: HttpServletRequest): Boolean =
        // Swagger / 정적 리소스는 JWT 검증 스킵 (성능 + 디버깅 편의)
        SWAGGER_PATHS.any { PATH_MATCHER.match(it, req.servletPath) }

    override fun doFilterInternal(
        req: HttpServletRequest,
        res: HttpServletResponse,
        chain: FilterChain,
    ) {
        val token = jwtTokenProvider.resolveToken(req)

        if (!token.isNullOrBlank()
            && jwtTokenProvider.isAccess(token)
            && jwtTokenProvider.validateToken(token)
        ) {
            val email = jwtTokenProvider.getEmail(token)
            val sid = jwtTokenProvider.getSessionId(token)          // 멀티 세션 / 강제 로그아웃 용

            // emailHash + softDelete 필터로 조회 — 평문 unique 가능성 회피
            val user = userQueryService.findActiveByEmail(email)

            if (user != null && email == user.email) {
                // (선택) 서버 측 세션 무효화 검증 — 강제 로그아웃 시
                if (sid != null && !userQueryService.isSessionActive(user.id, sid)) {
                    log.warn("invalidated session for user={}", user.id)
                } else {
                    val authorities = listOf<GrantedAuthority>(SimpleGrantedAuthority(user.role.name))
                    val auth = UsernamePasswordAuthenticationToken(user, null, authorities).apply {
                        details = WebAuthenticationDetailsSource().buildDetails(req)
                    }
                    SecurityContextHolder.getContext().authentication = auth
                }
            }
        }
        chain.doFilter(req, res)
    }
}
```

> **Kotlin idiom 포인트**:
> - `Objects.equals(a, b)` → `a == b` (Kotlin `==` 가 null-safe `equals`).
> - `StringUtils.hasText(token)` → `!token.isNullOrBlank()` (stdlib 으로 null-safe).
> - `apply { details = ... }` → Java 의 `var x = new ...(); x.setDetails(...)` 두 줄을 한 expression 으로.
> - `SecurityContextHolder.getContext().authentication = auth` — property syntax.

### 3.1 개선 사항 (원본 대비)

| 원본 | 개선 |
| --- | --- |
| `UserRepository` 직접 import (도메인 격리 X) | `UserQueryService` 통과 (도메인 / 인프라 격리) |
| `sid` claim 발급 후 검증 안 함 | 서버 측 세션 활성 검증 추가 |
| `logger.debug("...")` 평문 | SLF4J + 파라미터화 |
| `findByEmailHashAndDeletedAtIsNull` 노출 (인프라 누수) | service 메서드명 (`findActiveByEmail`) |
| Swagger path 가 코드에 hard-coded | companion object 의 `listOf` 분리 + AntPathMatcher 1회 생성 |

> **핵심**: 토큰 자체만 검증 → DB 까지 가서 사용자 살아있는지 / 권한 / 세션 확인. JWT stateless 의 한계를 보강.

---

## 4. `JwtAuthenticationEntryPoint` — 401 응답

```kotlin
// src/main/kotlin/com/example/shop/common/config/jwt/JwtAuthenticationEntryPoint.kt
@Component
class JwtAuthenticationEntryPoint(
    private val objectMapper: ObjectMapper,     // Bean 주입 (전역 Jackson 설정 활용)
) : AuthenticationEntryPoint {

    override fun commence(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authException: AuthenticationException,
    ) {
        val body = CommonResponse.fail<Unit>(
            ResponseCode.UNAUTHORIZED,
            "인증이 필요합니다.",
        )

        response.status = HttpServletResponse.SC_UNAUTHORIZED          // 401 (원본 버그 수정)
        response.contentType = MediaType.APPLICATION_JSON_VALUE
        response.characterEncoding = StandardCharsets.UTF_8.name()
        objectMapper.writeValue(response.outputStream, body)
    }
}
```

### 4.1 원본의 버그 (수정)

> ⚠️ **버그**: 원본 `job-answer-be` 의 `JwtAuthenticationEntryPoint` 는 `response.setStatus(HttpServletResponse.SC_BAD_REQUEST)` (400) — **401 응답해야 할 자리에 400 응답**. 클라이언트가 토큰 만료 / 미발급 케이스를 인증 실패로 못 잡음. **반드시 SC_UNAUTHORIZED (401)** 로 수정.

다른 개선:
- `ObjectMapper` 매번 새로 생성 → `@Bean` 주입 (한 번)
- `setCharacterEncoding(UTF-8)` 추가 (한글 메시지 깨짐 방지)
- `MediaType.APPLICATION_JSON_VALUE` 상수 사용

---

## 5. `JwtAccessDeniedHandler` — 403 응답 (신규)

```kotlin
// src/main/kotlin/com/example/shop/common/config/jwt/JwtAccessDeniedHandler.kt
@Component
class JwtAccessDeniedHandler(
    private val objectMapper: ObjectMapper,
) : AccessDeniedHandler {

    override fun handle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        ex: AccessDeniedException,
    ) {
        val body = CommonResponse.fail<Unit>(
            ResponseCode.INSUFFICIENT_PERMISSION,
            "해당 리소스에 접근할 권한이 없습니다.",
        )
        response.status = HttpServletResponse.SC_FORBIDDEN
        response.contentType = MediaType.APPLICATION_JSON_VALUE
        response.characterEncoding = StandardCharsets.UTF_8.name()
        objectMapper.writeValue(response.outputStream, body)
    }
}
```

→ 원본엔 없는 컴포넌트. `@PreAuthorize` 실패 / role 부족 시 일관된 envelope 응답.

---

## 6. `CorsProperties` — yaml 주입

```kotlin
// src/main/kotlin/com/example/shop/common/properties/CorsProperties.kt
@ConfigurationProperties(prefix = "cors")
data class CorsProperties(
    val allowedOrigins: String? = null,     // "https://shop.example.com,https://admin.example.com"
    val allowedMethods: String? = null,     // "GET,POST,PUT,PATCH,DELETE,OPTIONS"
    val allowedHeaders: String? = null,     // "*" 또는 명시
    val maxAge: Long? = null,                // 3600 (s)
)
```

> **Kotlin idiom 포인트**:
> - `@ConfigurationProperties` + `data class` + 생성자 binding — Spring Boot 가 자동으로 immutable bind.
> - Java 의 `@Getter @Setter` mutable POJO 가 immutable data class 로.
> - default `null` 로 모든 필드가 optional.

```yaml
# application.yml
cors:
  allowedOrigins: ${CORS_ORIGINS:http://localhost:3000}
  allowedMethods: GET,POST,PUT,PATCH,DELETE,OPTIONS
  allowedHeaders: Authorization,Content-Type,Accept-Language,X-DEVICE-UUID
  maxAge: 3600
```

```yaml
# application-prod.yml
cors:
  allowedOrigins: https://shop.example.com,https://admin.example.com
```

> **함정**: `*` (모든 origin 허용) + `setAllowCredentials(true)` 조합은 브라우저가 거부. JWT + Authorization 헤더는 `credentials = false` 권장 ([[../api-docs/swagger-springdoc]] 의 brower CORS 절 참고).

---

## 7. JwtTokenProvider — 표준 (jjwt 0.12)

```kotlin
// src/main/kotlin/com/example/shop/common/config/jwt/JwtTokenProvider.kt
@Component
class JwtTokenProvider(
    @Value("\${jwt.secret}") secret: String,
    @Value("\${jwt.access-minutes}") accessMinutes: Long,
    @Value("\${jwt.refresh-days}") refreshDays: Long,
    @Value("\${jwt.issuer:shop.example.com}") private val issuer: String,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    private val key: SecretKey
    private val accessTtl: Duration = Duration.ofMinutes(accessMinutes)
    private val refreshTtl: Duration = Duration.ofDays(refreshDays)

    init {
        // ⚠️ secret 은 Base64 권장. 256-bit 이상 필수.
        val keyBytes = Base64.getDecoder().decode(secret)
        require(keyBytes.size >= 32) {
            "JWT secret 은 256-bit (32 bytes) 이상이어야 합니다."
        }
        key = Keys.hmacShaKeyFor(keyBytes)
    }

    companion object {
        private const val CLAIM_ID    = "id"        // 사용자 PK
        private const val CLAIM_ROLE  = "role"
        private const val CLAIM_TYPE  = "token_type" // ACCESS / REFRESH
        private const val CLAIM_SID   = "sid"        // 세션 UUID

        const val AUTHORIZATION_HEADER = "Authorization"
        const val BEARER_PREFIX = "Bearer "

        private const val TYPE_ACCESS = "ACCESS"
        private const val TYPE_REFRESH = "REFRESH"
    }

    fun generateAccessToken(email: String, userId: Long, role: String, sessionId: String): String =
        build(email, userId, normalizeRole(role), TYPE_ACCESS, accessTtl, sessionId)

    fun generateRefreshToken(): String =
        build(null, null, null, TYPE_REFRESH, refreshTtl, null)

    private fun build(
        email: String?,
        userId: Long?,
        role: String?,
        type: String,
        ttl: Duration,
        sessionId: String?,
    ): String {
        val now = Instant.now()
        val builder = Jwts.builder()
            .issuer(issuer)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plus(ttl)))
            .claim(CLAIM_TYPE, type)
            .id(UUID.randomUUID().toString())             // jti
            .signWith(key, Jwts.SIG.HS256)

        if (type == TYPE_ACCESS) {
            email?.let { builder.subject(it) }
            userId?.let { builder.claim(CLAIM_ID, it) }
            role?.let { builder.claim(CLAIM_ROLE, it) }
            sessionId?.let { builder.claim(CLAIM_SID, it) }
        }
        // REFRESH 는 jti + iss + iat + exp 만. PII / role 없음.
        return builder.compact()
    }

    fun parseClaims(token: String): Claims =
        Jwts.parser()
            .verifyWith(key)
            .requireIssuer(issuer)
            .clockSkewSeconds(30)
            .build()
            .parseSignedClaims(token)
            .payload

    fun validateToken(token: String?): Boolean {
        if (token.isNullOrBlank()) return false
        return runCatching { parseClaims(token); true }
            .recoverCatching { ex ->
                when (ex) {
                    is ExpiredJwtException        -> log.debug("expired")
                    is JwtException               -> log.warn("invalid JWT: {}", ex.message)
                    is IllegalArgumentException   -> log.warn("empty JWT claims")
                    else                          -> throw ex
                }
                false
            }
            .getOrDefault(false)
    }

    fun isAccess(token: String): Boolean = runCatching {
        parseClaims(token).get(CLAIM_TYPE, String::class.java) == TYPE_ACCESS
    }.getOrDefault(false)

    fun isRefresh(token: String): Boolean = runCatching {
        parseClaims(token).get(CLAIM_TYPE, String::class.java) == TYPE_REFRESH
    }.getOrDefault(false)

    fun getEmail(token: String): String         = parseClaims(token).subject
    fun getJti(token: String): String           = parseClaims(token).id
    fun getUserId(token: String): Long?         = parseClaims(token).get(CLAIM_ID, Long::class.java)
    fun getRole(token: String): String          = normalizeRole(parseClaims(token).get(CLAIM_ROLE, String::class.java))
    fun getSessionId(token: String): String?    = parseClaims(token).get(CLAIM_SID, String::class.java)
    fun getExpiration(token: String): Instant   = parseClaims(token).expiration.toInstant()

    fun resolveToken(req: HttpServletRequest): String? {
        val bearer = req.getHeader(AUTHORIZATION_HEADER) ?: return null
        return if (bearer.startsWith(BEARER_PREFIX)) bearer.removePrefix(BEARER_PREFIX) else null
    }

    private fun normalizeRole(raw: String?): String {
        if (raw.isNullOrBlank()) return "ROLE_USER"
        return if (raw.startsWith("ROLE_")) raw else "ROLE_$raw"
    }
}
```

> **Kotlin idiom 포인트**:
> - `require(...)` — `IllegalArgumentException` throw 를 한 줄로.
> - `runCatching { } / .recoverCatching { }` — try-catch 체인을 expression chain 으로.
> - `?.let { }` — nullable 값에만 builder 추가 (`if (x != null)` 보다 짧음).
> - `bearer.removePrefix(BEARER_PREFIX)` — stdlib (substring 인덱스 계산 불필요).
> - `@Value("\${jwt.secret}")` — Kotlin string template 과 충돌 방지 (`\$` escape).
> - getter 들이 expression body — 한 줄.

### 7.1 개선 사항 (원본 대비)

| 원본 | 개선 |
| --- | --- |
| `secret.getBytes()` raw | `Base64.getDecoder().decode(secret)` 권장 + `require` 길이 검증 |
| issuer 검증 안 함 | `requireIssuer` + parse 시 검증 |
| clock skew 없음 | `clockSkewSeconds(30)` |
| `setIssuedAt`, `setExpiration` (옛 API) | 0.12 의 `issuedAt`, `expiration` |
| jti 를 `claim(CLAIM_JTI, ...)` 으로 직접 | jjwt 의 `id(...)` (표준 JWT id claim) |
| `getUserIdOrNull` 의 NumberFormatException 처리 | `Long::class.java` 로 직접 파싱 |
| `ExpiredJwtException` 핸들링 없음 | `runCatching` + `when` 분기 |

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET}                            # base64 인코드된 256-bit+ 시크릿. vault.
  access-minutes: 15                               # 짧게
  refresh-days: 14
  issuer: shop.example.com
```

---

## 8. ApiLogFilter — 모든 요청 추적

[[response-envelope#9]] 의 ApiLogFilter 패턴.

```kotlin
// src/main/kotlin/com/example/shop/common/filter/ApiLogFilter.kt
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
class ApiLogFilter(
    private val logs: ApiCallLogRepository,
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        req: HttpServletRequest,
        res: HttpServletResponse,
        chain: FilterChain,
    ) {
        val start = System.currentTimeMillis()
        try {
            chain.doFilter(req, res)
        } finally {
            val ex = req.getAttribute(ApiExceptionHandler.EXCEPTION_ATTR) as? Exception
            logs.recordAsync(
                ApiCallLog(
                    method     = req.method,
                    path       = req.requestURI,
                    query      = req.queryString,
                    status     = res.status,
                    durMs      = System.currentTimeMillis() - start,
                    clientIp   = req.clientIp(),
                    deviceUuid = req.getHeader("X-DEVICE-UUID"),
                    userId     = userIdFromContext(),
                    exType     = ex?.javaClass?.simpleName,
                    exMsg      = ex?.message?.take(500),
                ),
            )
        }
    }
}

/** X-Forwarded-For (LB / CDN) 우선, fallback 으로 remote addr. */
private fun HttpServletRequest.clientIp(): String =
    getHeader("X-Forwarded-For")
        ?.takeIf { it.isNotBlank() }
        ?.substringBefore(",")
        ?.trim()
        ?: remoteAddr

private fun userIdFromContext(): Long? =
    (SecurityContextHolder.getContext().authentication?.principal as? User)?.id
```

> **Kotlin idiom 포인트**:
> - `as? Exception` (safe cast) — null 또는 cast 성공 결과.
> - `message?.take(500)` — stdlib 으로 truncate.
> - extension `HttpServletRequest.clientIp()` — 헬퍼를 receiver 메서드로.
> - chain `?.takeIf { } ?.substringBefore(",") ?.trim() ?:` — null-safe pipeline.

운영 dashboard 에서:
- 5xx 비율 / endpoint 별 latency p95
- 같은 IP / device 의 짧은 시간 다회 요청 (brute force 의심)
- 특정 user 의 모든 action 추적

---

## 9. yaml 표준 세팅 (전체)

```yaml
# application.yml
spring:
  jackson:
    time-zone: UTC                             # 직렬화 시 UTC 일관
    serialization:
      WRITE_DATES_AS_TIMESTAMPS: false        # ISO-8601 문자열
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate

jwt:
  secret: ${JWT_SECRET}
  access-minutes: 15
  refresh-days: 14
  issuer: shop.example.com

cors:
  allowedOrigins: ${CORS_ORIGINS:http://localhost:3000}
  allowedMethods: GET,POST,PUT,PATCH,DELETE,OPTIONS
  allowedHeaders: Authorization,Content-Type,Accept-Language,X-DEVICE-UUID
  maxAge: 3600

logging:
  level:
    org.springframework.security: INFO
    com.example.shop: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: never                      # 운영 노출 X
```

---

## 10. 운영 체크리스트

- [ ] `JWT_SECRET` Base64 인코드 256-bit+. vault 주입.
- [ ] `accessMinutes ≤ 30` (보통 15)
- [ ] `refreshDays` 정책 명시 (보통 14~30)
- [ ] `corsProperties.allowedOrigins` 가 명시적 (와일드카드 X)
- [ ] `setAllowCredentials(false)` 또는 origin 와일드카드 X (둘 중 하나)
- [ ] `accessDeniedHandler` 존재 (403 일관 envelope)
- [ ] `authenticationEntryPoint` 가 **401 응답** (원본 버그 주의)
- [ ] 모든 public endpoint 명시 (whitelist), 그 외 `authenticated()`
- [ ] `actuator/prometheus` `actuator/health` 만 공개
- [ ] `BCryptPasswordEncoder` 또는 `argon2id` 사용 (강한 hash)
- [ ] CSRF disable + httpBasic disable + formLogin disable (stateless API 표준)
- [ ] 필터 순서 — ApiLog → Jwt → Username
- [ ] Swagger path 는 `shouldNotFilter` 에서 스킵 (성능)
- [ ] Kotlin DSL (`http { ... }`) 의 import 가 `org.springframework.security.config.annotation.web.invoke` (Java builder chain 과 혼용 X)

---

## 11. 함정 모음 (원본에서 발견 / 수정 + Kotlin 한정)

### 함정 1 — `JwtAuthenticationEntryPoint` 가 400 응답 ⚠️
원본 `job-answer-be` 버그: 401 자리에 `SC_BAD_REQUEST` (400). 클라이언트가 토큰 만료 분기 못 함. **`SC_UNAUTHORIZED`** 로 수정.

### 함정 2 — `setAllowCredentials(true)` + `*` origin
브라우저가 거부. JWT + Authorization 헤더면 `false` 가 정상.

### 함정 3 — JWT secret 이 짧음
`secret.getBytes()` 의 length 가 256-bit 미만이면 HMAC-SHA256 가 약함. **`require(keyBytes.size >= 32)`** 로 init 시 강제.

### 함정 4 — issuer 검증 안 함
다른 시스템 / 옛 토큰이 우연히 같은 secret 으로 서명되면 통과. **`requireIssuer`**.

### 함정 5 — clock skew 없음
서버 간 시간 1초만 어긋나도 만료 검증 미스. `clockSkewSeconds(30)`.

### 함정 6 — refresh token 에 PII 포함
원본은 refresh 에 email 안 넣지만, 일부 프로젝트는 실수로 넣음. **opaque (jti only)** + 서버 매핑.

### 함정 7 — `accessDeniedHandler` 미설정
403 응답이 default — envelope 깨짐. 직접 핸들러 등록.

### 함정 8 — filter 순서
`ApiLogFilter` 가 `JwtAuthenticationFilter` 보다 먼저 와야 인증 실패 케이스도 로그.

### 함정 9 — `permitAll()` path 누락
새 endpoint 추가 후 SecurityConfig 갱신 안 함 → 401. 매 endpoint 의 인증 정책을 PR 체크리스트.

### 함정 10 — `@PreAuthorize` 만 믿고 Service 단에서 권한 미체크
직접 호출 (배치 / 다른 서비스) 시 우회 가능. **Service / Domain 단에도 체크**.

### 함정 11 — Kotlin 한정: `@Value("\${...}")` 의 `$` escape
`@Value("${jwt.secret}")` 는 Kotlin string template 으로 해석돼 컴파일 에러. **`@Value("\${jwt.secret}")`** 로 escape.

### 함정 12 — Kotlin 한정: Java builder chain 과 DSL 혼용
`http.cors(c -> ...).csrf(c -> ...)` 식 Java builder 와 `http { cors { } csrf { } }` 식 Kotlin DSL 을 한 메서드에 섞으면 컴파일은 통과해도 의미가 어긋남. **둘 중 하나로 통일**, 신규 코드는 DSL 권장.

### 함정 13 — Kotlin 한정: `final class` 와 Spring proxy
`@Configuration` 클래스가 `final` (Kotlin default) 이면 CGLIB proxy 가 실패. **`kotlin-spring` plugin** 적용 → `@Component` / `@Configuration` 클래스가 자동 `open` 으로 컴파일됨.

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.spring") version "1.9.x"     // 또는 kotlin("plugin.allopen")
}
```

### 함정 14 — Kotlin 한정: data class 와 JPA Entity
이 문서의 `CorsProperties` 는 `data class` 이지만 JPA Entity 는 절대 `data class` 로 만들지 말 것 — `equals` / `hashCode` 가 모든 필드 기반이면 lazy collection 이 로딩됨. JPA Entity 는 일반 `class` + `open class` ([[../database/jpa]] 참고).

---

## 12. 관련

- [[response-envelope]] — 401/403/422 응답이 모두 이 envelope
- [[../api-docs/swagger-springdoc]] — Swagger 의 `@SecurityScheme(Bearer Authentication)`
- [[../api-design/login-jwt]] — 토큰 발급·갱신·로그아웃 흐름
- [[../pitfalls/concurrency-pitfalls]] (예정)
- [[../../java-spring/common/security-config|↗ Java 원본]]
