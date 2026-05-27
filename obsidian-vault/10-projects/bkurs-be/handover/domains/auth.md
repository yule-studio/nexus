---
title: "도메인 — auth (인증 / Account / JWT / Refresh)"
kind: handover
status: stable
created_at: 2026-05-21T23:40:00+09:00
tags: [bkurs-be, handover, domain, auth, jwt]
related:
  - "[[../../bkurs-be]]"
  - "[[../auth-and-security]]"
  - "[[../decisions/decision-account-single-table-role-status]]"
  - "[[../decisions/decision-auth-architecture-jwt-refresh]]"
  - "[[../decisions/decision-refresh-token-rotation-cookie]]"
home_hub: "[[../../bkurs-be]]"
---

# auth 도메인

> 회원가입 / 로그인 / Refresh rotation / JWT 발급 / 비밀번호 변경. 모든 인증 진입점은 본
> 패키지에서 시작 — 다른 도메인의 `@PreAuthorize` 가 본 도메인이 발급한 JWT 의 role 클레임을
> 의존한다. Security 설계 전체는 [[../auth-and-security]] 가 SSoT.

## 1. 책임

- 5 role (`MASTER` / `OWNER` / `TEACHER` / `STUDENT_HIGH` / `STUDENT_MIDDLE`) × 5 status 의 단일 계정 모델
- BCrypt password 저장 + 검증
- JWT HS256 access token 30분 발급
- Refresh token rotation (32-byte opaque, SHA-256 hash 저장, family 단위 reuse 감지)
- HttpOnly cookie 직렬화 (`/api/v1/auth` path 한정)

## 2. 핵심 엔티티

| 엔티티 | 파일 | 핵심 |
| --- | --- | --- |
| `Account` | `domain/Account.java` | id / email (uk) / passwordHash / role / status / createdAt / updatedAt — Builder 패턴 + `@PrePersist`/`@PreUpdate` |
| `Role` | `domain/Role.java` | enum 5종 |
| `AccountStatus` | `domain/AccountStatus.java` | `PENDING_CONNECT` → `PENDING_APPROVE` → `ACTIVE` / `REJECTED` / `BLOCKED` |
| `RefreshToken` | `domain/RefreshToken.java` | id / accountId / tokenHash (uk SHA-256) / familyId (UUID) / revoked / expiresAt / createdAt / revokedAt |

> Account 는 setter 없음 — 비밀번호 변경 / status 변경은 entity 메서드로. invariant 강제 위치.

## 3. 패키지 구성

```
auth/
├── controller/
│   ├── AuthController.java          # /api/v1/auth — signup/login/refresh/logout/me/change-password
│   └── RefreshCookieFactory.java    # HttpOnly Cookie 직렬화 (component)
├── service/                         # AccountService / RefreshTokenService / AuthService
├── domain/                          # Account / Role / AccountStatus / RefreshToken + Repository
├── dto/
│   ├── request/  SignupRequest, LoginRequest, ChangePasswordRequest
│   └── response/ TokenResponse, MeResponse
├── enums/                           # AuthErrorCode (AUT-)
├── exception/                       # InvalidCredentialsException, AccountAlreadyExistsException ...
├── handler/                         # AuthExceptionHandler — basePackages=com.bkurs.auth
└── jwt/                             # JwtTokenProvider / JwtAuthenticationFilter / JwtProperties
```

## 4. API endpoint

| Method | Path | 역할 | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/auth/signup` | (public) | 회원가입 — role 따라 status 초기값 결정 |
| POST | `/api/v1/auth/login` | (public) | 로그인 — BCrypt 검증 + Refresh 발급 |
| POST | `/api/v1/auth/refresh` | (public) | Refresh cookie 로 access 재발급 + rotation |
| POST | `/api/v1/auth/logout` | (public) | refresh revoke + cookie expire |
| GET | `/api/v1/auth/me` | (auth required) | 현재 사용자 (`@AuthenticationPrincipal Long accountId`) |
| POST | `/api/v1/auth/change-password` | (auth required) | 현재 비밀번호 검증 후 갱신 (PR #149) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V2 | `account` (uk email, role/status 32-char varchar) |
| V3 | `refresh_token` (uk token_hash, family_id, revoked, expires_at) |

## 6. 핵심 코드 흐름

### login

```
AuthController.login
  → AccountService.authenticate(email, rawPassword)
       → passwordEncoder.matches(raw, hash)
  → JwtTokenProvider.createAccessToken(account)   // 30분
  → RefreshTokenService.issueNewFamily(account)   // 새 family_id + DB INSERT
  → RefreshCookieFactory.issue(rawToken)          // HttpOnly Cookie
  → envelopes.ok(TokenResponse(accessToken, expiresIn), request) + Set-Cookie
```

### refresh rotation (B2B PR #6)

```
RefreshTokenService.rotate(rawToken)
  ▼
hash = tokenProvider.hashRefreshToken(rawToken)
existing = refreshTokenRepository.findByTokenHash(hash)
  ├── 미존재 / 만료     → InvalidCredentialsException (401)
  ├── revoked = true  → reuse 감지 → repository.revokeAllByFamilyId(existing.familyId) → throw
  │                    (이때 family revoke 는 noRollbackFor 로 commit 보장)
  └── 정상            → existing.revoke() + new row INSERT (same family_id) + new access token
```

## 7. JWT 클레임

`JwtTokenProvider.createAccessToken(account)`:

```java
Jwts.builder()
    .subject(String.valueOf(account.getId()))
    .claim("email", account.getEmail())
    .claim("role", account.getRole().name())      // ROLE_ prefix 는 filter 가 부여
    .claim("status", account.getStatus().name())
    .issuedAt(now)
    .expiration(now + 30m)
    .signWith(key)
    .compact();
```

`JwtAuthenticationFilter` 가 매 요청마다 parse → `SecurityContextHolder` 에 (principal=accountId,
authority=`ROLE_{role}`) 셋업. 자세히 [[../auth-and-security#3]].

## 8. 의존 / 의존받음

- **의존**: common.api / common.handler / Spring Security
- **의존받음 (caller)**: 거의 모든 도메인 — `@PreAuthorize` + `@AuthenticationPrincipal Long accountId`
- **cross-domain**: `AcademyMembershipResolver` (announcement/service) 가 role → academyId 매핑 시 본 도메인 Repository 참조

## 9. 회수 / pitfall

| 회수 | 해결 |
| --- | --- |
| ERROR dispatch `/error` 가 401 로 떨어짐 (B4A) | `GlobalValidationExceptionHandler` Order 높임 |
| `noRollbackFor` 미설정 시 reuse family revoke 가 롤백 (B2B) | `@Transactional(noRollbackFor = ReuseDetectedException.class)` |
| Bearer 누락 시 default 403 → 401 명시 | `SecurityConfig.exceptionHandling.authenticationEntryPoint(HttpStatusEntryPoint(UNAUTHORIZED))` |

## 서비스 로직 (`auth/service/`)

3 service class — orchestrator 패턴 (`AuthService` 가 `AccountService` + `RefreshTokenService` 묶음).

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `AccountService` | `@Transactional` | Account CRUD + BCrypt 검증 + password 변경 |
| `RefreshTokenService` | `@Transactional`, rotate 만 `noRollbackFor=ReuseException` | Refresh row issue / rotate / logout / reuse 감지 |
| `AuthService` | (orchestrator) | 위 두 service 묶어서 `signup / login / refresh / logout` 노출 |

### `AccountService` 주요 메서드

```java
public Account signup(String email, String rawPassword, Role role)        // email uk + BCrypt hash + status 초기값 (학생은 PENDING_CONNECT)
public Account verifyCredentials(String email, String rawPassword)         // 로그인 검증 — 실패 시 InvalidCredentialsException
public Account getById(Long id)
public void    changePassword(Long accountId, String oldPassword, String newPassword)
```

### `RefreshTokenService` 주요 메서드

```java
public IssuedRefreshToken issueForLogin(Account account, String userAgent, String ip)  // 새 family + INSERT
public RotateResult       rotate(String rawToken, String userAgent, String ip)         // hash lookup → revoked 면 reuse → family 전체 revoke
public void               logout(String rawToken)                                      // 단일 row revoke + cookie expire 신호
```

핵심 invariant: rotate 의 `@Transactional(noRollbackFor = RefreshTokenReuseException.class)` — reuse 감지 시 family revoke 가 commit 되어야 보안 효과.

### `AuthService` (orchestrator)

```java
public AuthResult signup(...)   // AccountService.signup → JWT 발급 → RefreshTokenService.issueForLogin
public AuthResult login(...)    // verifyCredentials → JWT → issueForLogin
public AuthResult refresh(...)  // RefreshTokenService.rotate → 새 access JWT
public void       logout(...)
```

전체 인증 흐름은 [[../modules/auth-and-security]] 참조.

## cross-link

- [[../auth-and-security]] — 전체 인증 흐름 SSoT
- [[academy]] — 가입 흐름의 연속 (PENDING_APPROVE → ACTIVE)
- [[../decisions/decision-account-single-table-role-status]]
- [[../decisions/decision-auth-architecture-jwt-refresh]]
- [[../decisions/decision-refresh-token-rotation-cookie]]
- [[../decisions/decision-cors-policy-allow-credentials]]
- [[../common]] — envelope / handler / RequestId
