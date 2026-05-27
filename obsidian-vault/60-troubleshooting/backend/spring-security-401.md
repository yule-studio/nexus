---
title: "troubleshooting · Spring Security 401 — Full authentication is required"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, spring-security, auth, jwt, cors]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[../network/network|network 트러블]]"
---

# Spring Security 401 — `Full authentication is required`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — SecurityFilterChain / CORS / JWT |

## 증상

```
{
  "timestamp": "2026-05-27T01:00:00Z",
  "status": 401,
  "error": "Unauthorized",
  "message": "Full authentication is required to access this resource",
  "path": "/api/orders"
}
```

- 평소 정상, Spring Boot upgrade (2.x → 3.x) 후 모든 endpoint 가 401
- 또는 일부 endpoint 만 401 (public 의도였는데)
- 또는 CORS preflight (OPTIONS) 가 401 → frontend 가 실제 호출 못함

## 가설

- API key 누락? → curl 에 Authorization 헤더 명시했는데 같은 결과
- 설정 누락? → `WebSecurityConfigurerAdapter` 가 deprecated 된 줄 몰랐음
- 실제: Spring Security 6.x (Spring Boot 3.x) 의 새 설정 방식 미적용

## 원인 (root cause)

Spring Security 5.7+ 에서 `WebSecurityConfigurerAdapter` deprecated, 6.x 에서 제거. 새 방식은 `SecurityFilterChain` bean 등록.

```java
// Spring Security 5.x — deprecated
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
            .antMatchers("/public/**").permitAll()
            .anyRequest().authenticated();
    }
}

// Spring Security 6.x — 새 방식
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }
}
```

Spring Boot 3.x 업그레이드 시 옛 config 가 그대로 있으면 default deny → 모두 401.

## 해결

### 즉시 — 옛 설정을 새 방식으로 교체

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())            // REST API 라면 (세션 인증이면 켜기)
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()   // ← preflight 허용
                .requestMatchers("/health", "/actuator/health/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))   // JWT
            .exceptionHandling(e -> e
                .authenticationEntryPoint(authEntryPoint())
                .accessDeniedHandler(accessDeniedHandler())
            )
            .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration c = new CorsConfiguration();
        c.setAllowedOrigins(List.of("https://app.example.com"));
        c.setAllowedMethods(List.of("GET","POST","PUT","DELETE","PATCH","OPTIONS"));
        c.setAllowedHeaders(List.of("Authorization","Content-Type"));
        c.setAllowCredentials(true);
        c.setMaxAge(3600L);

        var src = new UrlBasedCorsConfigurationSource();
        src.registerCorsConfiguration("/**", c);
        return src;
    }
}
```

### 자주 만나는 4가지 함정

**(1) CORS preflight (OPTIONS) 401**

브라우저는 actual request 전에 OPTIONS preflight 발진. Security 가 OPTIONS 도 인증 요구하면 차단.

```java
.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
```

또는 CORS 설정이 SecurityFilterChain 보다 앞에 적용되도록:
```java
.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

**(2) JWT signature verification 실패**

```
JwtValidationException: An error occurred while attempting to decode the Jwt:
  Signed JWT rejected: Invalid signature
```

원인: issuer-uri 의 JWKs 와 token signing key 불일치.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/myrealm
          # 또는
          jwk-set-uri: https://auth.example.com/realms/myrealm/protocol/openid-connect/certs
```

token 의 `iss` claim 과 issuer-uri 정확히 일치해야 함.

**(3) `permitAll()` 가 작동 안 함**

```java
// Bad — order 가 잘못
.anyRequest().authenticated()
.requestMatchers("/public/**").permitAll()   // ← 위에서 이미 authenticated 매칭
```

`anyRequest()` 는 항상 마지막. matcher 는 specific → general 순서.

**(4) `H2 console` / Actuator 차단**

```java
.requestMatchers("/h2-console/**").permitAll()
.headers(h -> h.frameOptions(f -> f.sameOrigin()))   // H2 가 iframe 사용
```

```java
// Actuator
.requestMatchers("/actuator/health/**", "/actuator/info").permitAll()
.requestMatchers("/actuator/**").hasRole("ADMIN")
```

## CSRF 의 함정

```java
// REST API stateless
.csrf(csrf -> csrf.disable())

// Session 기반 web app — CSRF 켜기 (default)
.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .ignoringRequestMatchers("/api/webhooks/**")   // webhook 제외
)
```

POST/PUT/DELETE 가 403 CSRF 거부 → 위 패턴 확인.

## 진단

```bash
# curl 로 직접 확인
curl -v https://api.example.com/orders \
  -H "Authorization: Bearer eyJ..."
# → 응답 헤더 / WWW-Authenticate 확인

# JWT decode
echo "eyJhbGc..." | cut -d. -f2 | base64 -d | jq

# Spring Boot endpoint
curl localhost:8080/actuator/mappings | jq '.contexts.application.mappings.dispatcherServlets'
# → 어떤 URL 이 어떤 controller 에 매핑되는지

# Filter chain 로그
logging.level.org.springframework.security: DEBUG
logging.level.org.springframework.web: DEBUG
```

→ 로그에서 어느 filter 가 401 반환했는지 추적.

## 재발 방지

- **회귀 테스트**: `@WebMvcTest` 또는 `@SpringBootTest` + `MockMvc`
  ```java
  @Test
  void publicEndpointShouldBeAccessibleWithoutAuth() throws Exception {
      mockMvc.perform(get("/api/public/health"))
          .andExpect(status().isOk());
  }

  @Test
  void protectedEndpointShouldReturn401WithoutToken() throws Exception {
      mockMvc.perform(get("/api/orders"))
          .andExpect(status().isUnauthorized());
  }

  @Test
  @WithMockUser(roles = "USER")
  void protectedEndpointShouldReturn200WithToken() throws Exception {
      mockMvc.perform(get("/api/orders"))
          .andExpect(status().isOk());
  }

  @Test
  void corsPreflightShouldBeAllowed() throws Exception {
      mockMvc.perform(options("/api/orders")
          .header("Origin", "https://app.example.com")
          .header("Access-Control-Request-Method", "POST"))
          .andExpect(status().isOk());
  }
  ```
- **upgrade 시 deprecation note 확인** — Spring Boot Release Notes 의 Security 섹션 필독
- **명시적 `permitAll()`** — public endpoint 는 한 곳에 모아 documented
- **CI 단계 보안 스캔**: OWASP ZAP / Trivy

## 관련

- [[backend]]
- [[../network/network|network 트러블]] — CORS / TLS 영역
- [[../../20-areas/backend/spring/spring|20-areas/spring]]
- 책: Spring Security in Action 2nd Ed
