---
title: "B2B — Refresh 회전 + HttpOnly Cookie + logout + CORS 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b2-auth
created_at: 2026-05-20T04:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-refresh-token-rotation-cookie]]"
  - "[[../decisions/decision-cors-policy-allow-credentials]]"
  - "[[task-log-b2a-account-jwt]]"
tags: [task-log, auth, refresh-token, cookie, cors, b2b]
---

# 작업

GitHub issue [#5](https://github.com/seum-running/bkurs-be/issues/5) — Refresh 토큰 회전 + HttpOnly Cookie + logout + CORS. B2A 의 access only 한계를 보완.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| refresh 저장 | DB 테이블 `refresh_token` + SHA-256 hash (평문 미보유) | [[../decisions/decision-refresh-token-rotation-cookie]] |
| rotation | refresh 사용 시마다 직전 row revoke(ROTATION) + 새 row insert | OWASP 권고 |
| reuse detection | revoked token 재사용 시 family 전체 revoke(REUSE_DETECTED) + 401 | 탈취 감지 시 정상 클라도 함께 logout → 사용자가 재로그인하며 인지 |
| cookie | HttpOnly + SameSite=Lax + Path=/api/v1/auth + Secure(prod) | XSS 차단 + CSRF 1차 + 다른 endpoint 비대화 방지 |
| refresh 형태 | opaque random (32B base64url) | JWT 아님 — DB lookup 으로 검증하므로 self-contained 클레임 불필요 |
| 만료 | 7일 (604800000ms), rotation 시 갱신 | 학원 사용자 주간 사용 패턴 가정 |
| CORS | `CorsConfigurationSource` bean + `allowedOriginPatterns` + allow-credentials=true | bkurs-fe (localhost:5173) cross-origin + cookie 송수신 |

# 진행

## 2026-05-20

### 1. branch / issue

- B2A merge 대기 중에 stacked PR 패턴으로 진행 — branch base=feature/issue-3-b2a-account-jwt
- issue #5 생성 (assignee codwithyc, label enhancement, audit 섹션에 4 대안 비교 두 결정 노트 링크)
- decision 2개 작성 (refresh-token-rotation-cookie / cors-policy-allow-credentials)

### 2. 코드 (commit 단위)

- C1 (a510967): RefreshToken entity + Repository + V3 마이그레이션. token 평문 미보유, RevokeReason enum (ROTATION/LOGOUT/REUSE_DETECTED/ADMIN), bulk update revokeFamily/revokeAllByAccount
- C2 (a89679a): RefreshTokenService + JwtTokenProvider 확장 + jwt/cors application.yml. generateRefreshToken(SecureRandom 32B base64url) + hashRefreshToken(SHA-256 hex)
- C3 (d26ae92): AuthController refresh/logout + HttpOnly cookie + CORS. RefreshCookieFactory(issue/expire) + AuthService.AuthResult record + SecurityConfig.cors(withDefaults()) + permitAll 추가
- C4 (955c848): rotate() 에 @Transactional(noRollbackFor = RefreshTokenReuseException.class) — reuse 감지 시 family revoke commit 보존 (회귀 테스트 디버깅 중 발견)
- C5 (8a6550c): RefreshTokenApiTest 6 회귀 + CorsConfigTest preflight + CHANGELOG B2B 섹션

### 3. PR

- PR [#6](https://github.com/seum-running/bkurs-be/pull/6) (feature/issue-5 → feature/issue-3-b2a-account-jwt, stacked) open
- CI: 진행 중 (watch)
- 머지: B2A (#4) 머지 후 GitHub 가 base 를 dev 로 자동 retarget — 사용자 manual merge

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| `RefreshToken` 별도 테이블 (account 분리) | refresh 는 1 account : N tokens (multi-device / rotation chain). account row 에 묻으면 N 차원이 무너짐. | role 별 정책 (강사 device 제한 등) 은 B3+ 에서 별도 컬럼. |
| `token_hash` UNIQUE | reuse detection 의 핵심 — 같은 hash 로 2회 요청은 즉시 family revoke 표적이 된다. | hash 가 평문 token 의 SHA-256 이라 DB 유출 시에도 reverse 불가. |
| `family_id` UUID | rotation 체인 추적 — 탈취 의심 시 같은 family 만 revoke (다른 device 의 family 는 영향 없음). | account-wide revoke 는 `revokeAllByAccount` 별도 메서드. |
| `RevokeReason` enum 4종 | 감사 / 운영 화면에서 "정상 회전 / 사용자 logout / 탈취 의심 / 관리자 강제" 구분. | string 이 아닌 enum 으로 둬서 typo 방지 + 검색 인덱스 효율. |
| refresh = opaque random (NOT JWT) | DB lookup 으로 검증하므로 self-contained 클레임 불필요. JWT 면 토큰 자체에 사용자 정보 노출. | RS256 비대칭 JWT 대안은 다중 서비스 분리 시. |
| HS256 secret + SHA-256 hash (둘 다 application yml 의 jwt.secret 무관) | hash 는 키 없는 단방향 → secret rotation 영향 없음. JWT secret 회전 시 access 만 재발급, refresh DB row 영향 없음. | HMAC-SHA256 hash 로 추가 보안 가능 — 7일 사이클이라 단순 SHA-256 충분. |
| Cookie `HttpOnly + SameSite=Lax + Path=/api/v1/auth` | XSS 면역 + CSRF 1차 방어 + 다른 endpoint 요청에서 cookie 자동 전송 안 됨 (헤더 비대화 방지). | LMS 임베드 (cross-site iframe) 요구 발생 시 `SameSite=None` 으로 조정. |
| `RefreshCookieFactory` 별도 클래스 | controller / service 가 cookie 직렬화 모름. yml 의 정책 변경이 한 클래스만 영향. | profile 별 정책 (prod Secure=true) 도 본 클래스가 흡수. |
| `AuthService.AuthResult` record | TokenResponse(JSON body) + refreshTokenPlain(cookie) 두 채널을 controller 가 분리 직렬화. | service 가 controller-specific 응답 만들지 않고 transport agnostic 유지. |
| `noRollbackFor = RefreshTokenReuseException` | RuntimeException default rollback 으로 family revoke 가 묻혀 보안 의미 무력화됨. | 부수효과 commit + 예외 응답 둘 다 필요한 드문 케이스에 정확한 패턴 — vault 회수 노트 후보. |
| CORS `allowedOriginPatterns` (NOT `allowedOrigins`) | Spring 6 / 모던 브라우저 모두 `*` + `allow-credentials=true` 거부. patterns 는 prod 서브도메인 wildcard 허용. | `https://*.bkurs.kr` 로 prod 학원별 서브도메인에 강함. |
| `CorsConfigurationSource` bean + `SecurityConfig.cors(withDefaults())` | WebMvc + Spring Security 가 같은 source 참조 → SSoT. | filter 직접 작성 / WebMvcConfigurer addCorsMappings 는 중복. |
| `permitAll` 에 `/refresh` `/logout` 추가 | cookie 기반 검증이라 Bearer 불필요 — Spring Security filter 가 401 차단하면 endpoint 가 cookie 못 봄. | controller 안에서 cookie 검증 + 401 직접 throw. |
| CHANGELOG `noRollbackFor` 항목 명시 | 후속 개발자가 "왜 이 어노테이션이 있나" 검색 가능. | vault `20-areas/.../pitfalls/transactional-rollback-noRollbackFor.md` 추가 후보. |

# 다음 단계 (예정)

- ~~PR 생성 → CI watch~~
- B2A (#4) merge → B2B (#6) base 자동 retarget → CI 재실행 (gh 가 자동) → 사용자 merge
- vault 본 task-log status: in-progress → complete (사용자 merge 후)
- B2C 시작 — 학원 코드 입력 endpoint + 학원 승인/거부 + 상태 전이 (PENDING_CONNECT → PENDING_APPROVE → ACTIVE)

# 회수 / pitfall (누적)

- @Transactional default rollback 이 부수효과(family revoke)를 묻어버리는 케이스 — RefreshTokenReuseException 의 noRollbackFor 필요. 디버깅 1 cycle 소요. vault `20-areas/backend/spring/java-spring/pitfalls/transactional-rollback-noRollbackFor.md` 추가 후보.
- 격리 안 된 통합 테스트가 의도치 않은 fixture (signup auto-issue refresh) 로 실패. 명시적 `refreshTokenRepository.deleteAll()` + helper `onlyFamilyId()` 패턴.

# 참고

- [[../decisions/decision-refresh-token-rotation-cookie]]
- [[../decisions/decision-cors-policy-allow-credentials]]
- [[task-log-b2a-account-jwt]]
- OWASP Token Storage / Refresh Rotation cheat sheet
