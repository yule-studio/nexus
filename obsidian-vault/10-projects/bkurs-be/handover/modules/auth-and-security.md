---
title: "bkurs-be — 인증 / 보안 / CORS 인수인계"
kind: handover
status: stable
created_at: 2026-05-21T22:20:00+09:00
tags: [bkurs-be, handover, auth, security, jwt, cors]
related:
  - "[[../bkurs-be]]"
  - "[[overview]]"
  - "[[common]]"
  - "[[decisions/decision-auth-architecture-jwt-refresh]]"
  - "[[decisions/decision-refresh-token-rotation-cookie]]"
  - "[[decisions/decision-cors-policy-allow-credentials]]"
home_hub: bkurs-be
---

# bkurs-be — 인증 / 보안 / CORS

> JWT + Refresh rotation + HttpOnly Cookie + CORS allow-credentials + `@PreAuthorize` role
> matrix. **이걸 안 보고 auth/security 코드 만지면 401/403 회귀 거의 확실.**

## 1. 한 장 요약

```
Client                                                  Server
  │                                                       │
  │  POST /api/v1/auth/login  { email, password }          │
  │ ─────────────────────────────────────────────────────▶ │
  │                                                       │ Account.match(password) → BCrypt 검증
  │                                                       │ JwtTokenProvider.createAccessToken(account)
  │                                                       │ RefreshTokenService.issue(account) → family
  │  200 { data: { accessToken, expiresIn } }              │
  │  Set-Cookie: bkurs_rt=<opaque>; HttpOnly; Path=/api/v1/auth; SameSite=Lax
  │ ◀───────────────────────────────────────────────────── │
  │                                                       │
  │  GET /v1/students/me/homework                          │
  │  Authorization: Bearer <access>                        │
  │ ─────────────────────────────────────────────────────▶ │
  │                                                       │ JwtAuthenticationFilter:
  │                                                       │   - parse / verify
  │                                                       │   - SecurityContext = (accountId, ROLE_*)
  │                                                       │ @PreAuthorize("hasRole('STUDENT_HIGH')")
  │  200 envelope ...                                      │
  │ ◀───────────────────────────────────────────────────── │
  │                                                       │
  │ (access 만료 30분 후)                                  │
  │  POST /api/v1/auth/refresh                             │
  │  Cookie: bkurs_rt=<opaque>                             │
  │ ─────────────────────────────────────────────────────▶ │
  │                                                       │ hash(token) lookup → DB row
  │                                                       │   - revoked / family revoked → 401 (reuse 감지)
  │                                                       │   - valid → rotate (new token, old.revoked=true)
  │  200 { data: { accessToken } }                         │
  │  Set-Cookie: bkurs_rt=<new>; ...                       │
  │ ◀───────────────────────────────────────────────────── │
```

## 2. Account 모델

`Account` (단일 테이블, V2):

| 컬럼 | 값 |
| --- | --- |
| `id` | PK |
| `email` | unique |
| `password_hash` | BCrypt |
| `role` | enum (`MASTER` / `OWNER` / `TEACHER` / `STUDENT_HIGH` / `STUDENT_MIDDLE`) |
| `status` | enum (`ACTIVE` / `PENDING` / `BLOCKED` / `SUSPENDED` / `DELETED`) |
| `created_at` / `updated_at` | |

**Role 5종 + Status 5종**. 검토 alternatives 는 [[decisions/decision-account-single-table-role-status]].

## 3. JWT Access Token (`auth/jwt/JwtTokenProvider`)

- 알고리즘: **HS256 (대칭키)**. 단일 서비스라 충분 — 다중 서비스 분리 시 RS256 검토.
- 만료: 30분 (`jwt.access-token-expiration-ms: 1800000`).
- secret: `jwt.secret` env (32 bytes 이상). local default 는 `bkurs-local-dev-secret-do-not-use-in-prod-change-me-please`. prod 는 secret manager.
- claims:

| claim | 값 |
| --- | --- |
| `sub` | accountId (long) |
| `email` | account.email |
| `role` | Role enum name |
| `status` | AccountStatus enum name (BLOCKED 검증용) |
| `iat` / `exp` | 표준 |

### 3.1 `JwtAuthenticationFilter`

`OncePerRequestFilter` extends. 매 요청:

1. `Authorization: Bearer <token>` 추출
2. `tokenProvider.parse(token)` → JwtException 시 SecurityContext 비움 (anonymous 처리, 401 은 SecurityConfig)
3. principal = accountId (Long), authority = `ROLE_{Role.name()}` (예: `ROLE_STUDENT_HIGH`)
4. `SecurityContextHolder.getContext().setAuthentication(new UsernamePasswordAuthenticationToken(...))`

### 3.2 controller 에서 현재 사용자 추출

```java
@GetMapping("/me")
public ResponseEntity<ApiEnvelope<MeResponse>> me(@AuthenticationPrincipal Long accountId,
                                                  HttpServletRequest req) { ... }
```

## 4. Refresh Token (`auth/service/RefreshTokenService`)

별도 테이블 `refresh_token` (V3):

| 컬럼 | 값 |
| --- | --- |
| `id` | PK |
| `account_id` | FK |
| `token_hash` | SHA-256 hex (64 chars) — **token 평문은 cookie 만** |
| `family_id` | UUID — reuse 감지용 그룹 |
| `revoked` | bool |
| `expires_at` | login + 7일 |
| `created_at` / `revoked_at` | |
| uk(`token_hash`) | |

만료: 7일 (`jwt.refresh-token-expiration-ms: 604800000`).

### 4.1 rotation 흐름

`/api/v1/auth/refresh`:

1. cookie 의 raw token → `hashRefreshToken(token)` (SHA-256 hex)
2. DB lookup
3. **만료 / revoked → 401**
4. **유효 → rotate**
   - 새 raw token 생성 (32-byte base64url, 47 chars)
   - 새 row INSERT (`family_id = old.family_id`)
   - old row UPDATE `revoked = true`
   - new access token 발급
   - 응답: `Set-Cookie: bkurs_rt=<new>` + `{ accessToken }`

### 4.2 reuse 감지

이미 `revoked = true` 인 row 가 다시 들어옴 (탈취 의심) → 같은 `family_id` 의 **모든 row revoke**.

> ⚠️ `noRollbackFor`: family revoke 트랜잭션은 reuse 발견 시 throw 직전 commit. 자세히
> [[decisions/decision-refresh-token-rotation-cookie]].

### 4.3 logout

`/api/v1/auth/logout`: 현재 cookie 의 token revoke + `Set-Cookie: bkurs_rt=; Max-Age=0`.

## 5. Refresh Cookie (`auth/controller/RefreshCookieFactory`)

- **이름**: `jwt.refresh-cookie-name = bkurs_rt`
- **HttpOnly**: XSS 차단
- **Secure**: `jwt.refresh-cookie-secure` (local=false, prod=true)
- **SameSite**: Lax (CSRF 1차 방어 + same-site 네비게이션 호환)
- **Path**: `/api/v1/auth` — refresh/logout 만 cookie 송신, 다른 endpoint 에 새지 않음
- **Max-Age**: refresh 만료와 동일 (7일)

> SameSite=Strict 가 아닌 이유: 외부 링크 → 자동 로그인 유지 불가. Lax 가 UX/보안 균형.

## 6. SecurityConfig

```
- stateless (SessionCreationPolicy.STATELESS)
- csrf disabled (JWT + SameSite=Lax)
- httpBasic / formLogin / logout 비활성
- cors.withDefaults() — CorsConfig bean 자동 참조
- permitAll: /api/v1/auth/{signup,login,refresh,logout}
              /api/v1/health, /actuator/{health,info}
              /swagger-ui.html, /swagger-ui/**, /v3/api-docs, /v3/api-docs/**
- anyRequest().authenticated()
- exceptionHandling.authenticationEntryPoint = 401 명시 (default 403 와 구분)
- JwtAuthenticationFilter 를 UsernamePasswordAuthenticationFilter 앞에
- @EnableMethodSecurity — @PreAuthorize / @PostAuthorize 활성
```

코드: [`SecurityConfig`](file:src/main/java/com/bkurs/common/config/SecurityConfig.java).

## 7. CORS (`common/config/CorsConfig`)

| 키 | 값 |
| --- | --- |
| `cors.allowedOriginPatterns` | `https://*.bkurs.kr` (prod) / `http://localhost:5173` (local) — **patterns 사용**, `allowedOrigins` 가 아님 |
| `cors.allowedMethods` | GET POST PUT DELETE PATCH OPTIONS |
| `cors.allowedHeaders` | `*` |
| `cors.exposedHeaders` | (지정) — `X-Request-Id` 등 |
| `cors.allowCredentials` | **true** — cookie 동봉 필수 |
| `cors.maxAge` | 3600 |

> **`allowedOrigins=*` + `allowCredentials=true` 조합은 Spring 6 / 모던 브라우저 금지.**
> 반드시 `allowedOriginPatterns` 사용. 자세히 [[decisions/decision-cors-policy-allow-credentials]].

## 8. `@PreAuthorize` role matrix

```java
@PreAuthorize("hasRole('OWNER')")                    // 학원장
@PreAuthorize("hasRole('TEACHER')")                  // 강사
@PreAuthorize("hasRole('STUDENT_HIGH')")             // 고등 학생
@PreAuthorize("hasRole('STUDENT_MIDDLE')")           // 중등 학생
@PreAuthorize("hasAnyRole('STUDENT_HIGH','STUDENT_MIDDLE')")  // 학생 공통
@PreAuthorize("hasRole('MASTER')")                   // 본사
```

> Spring Security 가 `ROLE_` prefix 자동 부여 → `hasRole('OWNER')` 는 `ROLE_OWNER` 매칭.

### 8.1 path role prefix 와 1:1

| path | 요구 role | controller prefix |
| --- | --- | --- |
| `/v1/students/me/...` | STUDENT_HIGH 또는 STUDENT_MIDDLE | `Student*Controller` |
| `/v1/teachers/me/...` | TEACHER | `Teacher*Controller` |
| `/v1/academies/me/...` | OWNER | `Owner*Controller` |
| `/v1/master/...` `/v1/admin/...` | MASTER | `Master*Controller` / `Admin*Controller` |
| `/api/v1/auth/...` | permitAll (signup/login/refresh/logout) 또는 authenticated (me/change-password) | `AuthController` |

규칙을 깨면 `@PreAuthorize` 와 path 가 어긋남 → 401/403 회귀 발생.

## 9. 권한 2계층 검증 (`AcademyMembershipResolver`)

role 만으로 부족한 경우 — "이 학생이 정말 이 학원 소속인가" 등 cross-domain.

```java
@PreAuthorize("hasRole('OWNER')")
public List<X> ownerList(@PathVariable long academyId, @AuthenticationPrincipal Long accountId) {
    Academy academy = academyService.getMine(accountId);   // 1계층 — role
    if (academy.getId() != academyId) throw new ForbiddenException(...);  // 2계층 — ownership
    return service.listForAcademy(academyId);
}
```

`AcademyMembershipResolver` (announcement/service) — role → academyId 통합 매핑. announcement /
dashboard 가 의존.

## 10. 인증 회수 / pitfall

| 회수 | 어디서 | 해결 |
| --- | --- | --- |
| ERROR dispatch `/error` 가 401 로 떨어짐 | B4A | `GlobalValidationExceptionHandler` 추가 (Order 더 높임) |
| `noRollbackFor` 미설정 시 reuse 감지 family revoke 가 롤백 | B2B | `@Transactional(noRollbackFor = ReuseDetectedException.class)` |
| `STUDENT 거부` 케이스 401 / 403 둘 다 나옴 | B7E | Spring 6 환경 차이. 테스트는 둘 다 허용 |
| CORS allowedOrigins + credentials 동시 → 차단 | B2B | `allowedOriginPatterns` 로 전환 |
| Bearer 누락 시 default 403 → 클라이언트 혼란 | B2A | `HttpStatusEntryPoint(UNAUTHORIZED)` 명시 |

## 11. password / signup

- `BCryptPasswordEncoder` (cost 10 default). 변경: bean override.
- signup 시 `Account.status = PENDING` 가능 (학원 connect 흐름) — 자세히 [[decisions/decision-academy-connect-membership-flow]].
- `change-password` PR #149 — 별도 endpoint, current password 검증 + new hash 저장.

## 12. 토큰 회피 / blocklist (현재 없음)

- access token 강제 무효화 없음 (만료 대기). 긴급 시:
  1. `Account.status = SUSPENDED` 로 변경
  2. `JwtAuthenticationFilter` 가 매 요청 status BLOCKED 검증 (현재 미구현 — 후속)
- refresh token revoke 는 즉시 (DB row update).

## cross-link

- [[../bkurs-be]] — 프로젝트 hub
- [[overview]] — FilterChain / 7 layer
- [[common]] — handler / envelope
- [[decisions/decision-auth-architecture-jwt-refresh]]
- [[decisions/decision-refresh-token-rotation-cookie]]
- [[decisions/decision-cors-policy-allow-credentials]]
- [[decisions/decision-account-single-table-role-status]]
- [[decisions/decision-teacher-permission-matrix]] — 강사 한정 권한
