---
title: "B2A — Account entity + JWT 회원가입/로그인 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b2-auth
created_at: 2026-05-20T01:00:00+09:00
agent: engineering-agent/tech-lead
status: complete
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-auth-architecture-jwt-refresh]]"
  - "[[../decisions/decision-account-single-table-role-status]]"
tags: [task-log, auth, jwt, b2a]
---

# 작업

GitHub issue [#3](https://github.com/seum-running/bkurs-be/issues/3) — Account entity + JWT 회원가입 / 로그인 / me.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 인증 방식 | JWT Access + HttpOnly Refresh Cookie 하이브리드 | [[../decisions/decision-auth-architecture-jwt-refresh]] |
| Account 테이블 | Single Table + Role enum (5) + Status enum (5) + Profile 분리 | [[../decisions/decision-account-single-table-role-status]] |
| 비번 해싱 | BCrypt (Spring Security 기본 BCryptPasswordEncoder, strength 10) | strength 10 = 약 100ms/hash (BCrypt default) — security ↔ UX 균형 |
| Access Token 만료 | 30분 | UX vs 보안 균형. refresh 가 7일이라 사용자 재로그인 빈도 적음 |
| Token 서명 | HS256 (대칭키, application yml 의 jwt.secret) | RS256 (비대칭) 은 다중 서비스 분리 시 사용 — bkurs-be 단일 서비스라 HS256 충분 |
| 회원가입 default status | PENDING_CONNECT (학생) / ACTIVE (강사·원장은 학원 측 inviite — 별도 endpoint) | 학생은 가입 직후 학원 코드 입력 필요. 본 PR 은 학생만, 강사·원장은 B3 |

# 진행

## 2026-05-20

### 1. branch / issue
- dev pull → `feature/issue-3-b2a-account-jwt` 분기
- issue #3 생성 (assignee codwithyc, label enhancement)

### 2. decision 노트
- [[../decisions/decision-auth-architecture-jwt-refresh]] — JWT + Refresh 하이브리드 + 4 대안 비교 + Why 4 구조
- [[../decisions/decision-account-single-table-role-status]] — Single Table + Profile 분리 + 4 대안 비교 + 인덱스 설계

### 3. 코드 작업 (commit 단위)

- C1: build.gradle (jjwt 0.12.x 의존성) + Account entity + Role / AccountStatus enum + V2__create_account.sql
- C2: AccountRepository + AccountService (BCrypt + 중복 이메일 검증)
- C3: JwtTokenProvider + JwtAuthenticationFilter + SecurityConfig 강화 + jwt.* application.yml 설정
- C4: AuthController + DTO (SignupRequest / LoginRequest / TokenResponse / AccountResponse) + AuthService (signup → ACTIVE 외 status 분기)
- C5: AuthExceptionHandler (@RestControllerAdvice, base = com.bkurs.auth) — `InvalidCredentialsException` → 401 / `DuplicateEmailException` → 409. SecurityConfig 에 `HttpStatusEntryPoint(UNAUTHORIZED)` 등록.
- C6: AuthApiTest 5종 통합 회귀 (happy / teacher ACTIVE / 잘못된 비번 401 / Bearer 없음 401 / invalid Bearer 401) + CHANGELOG B2A 마무리.

### 4. PR

- PR [#4](https://github.com/seum-running/bkurs-be/pull/4) (feature/issue-3 → dev) open
- CI: build + test (Java 21, mysql:9 service) — 회귀 9 GREEN
- 머지 결과: (CI 완료 후 기록)

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| `Account` 단일 테이블 | role 별로 테이블 분리하면 join 비용 + 권한 체크가 분산. 5 role 모두 동일 인증 흐름 → 단일 테이블이 read path 최적. | role 별 profile 분리는 학생/강사 식별자(학년, 학원ID)가 다를 때 별도 테이블로. B3+. |
| `AccountStatus` 5종 enum | `PENDING_CONNECT` (학생 가입 직후) → `PENDING_APPROVE` (학원 코드 입력 후 승인 대기) → `ACTIVE`. 강사/원장은 학원 측 invite 기반이므로 `ACTIVE` 즉시. | `BLOCKED` 는 정책 위반, `REJECTED` 는 학원이 승인 거부. 모두 운영 정책 분기에 사용. |
| `transitionTo()` 불변식 메서드 | enum 값 직접 mutation 허용하면 잘못된 상태 전이 (예: REJECTED → ACTIVE) 가능. 도메인 메서드로 강제. | 상태 머신 라이브러리(Spring Statemachine) 는 oversized — 5 상태에 라이브러리 도입 가치 없음. |
| JWT HS256 + access only (B2A) | 단일 서비스 → 대칭키 충분. RS256 은 다중 서비스 분리 시. access 30분은 UX 균형. | refresh / logout / CORS 는 B2B. 만료 30분이라 토큰 탈취 영향 제한. |
| `JwtAuthenticationFilter` principal = `Long accountId` | `@AuthenticationPrincipal Long` 으로 controller 가 즉시 DB lookup 가능. `UserDetails` 객체 캐싱은 stateless 와 충돌. | role 권한은 `ROLE_{Role.name()}` authority 로 매핑 → `@PreAuthorize("hasRole('OWNER')")` 호환. |
| `HttpStatusEntryPoint(UNAUTHORIZED)` | Spring Security default 인증 실패가 403 → 권한 실패(403, B3+) 와 의미 충돌. 클라(bkurs-fe) 가 401 기준으로 토큰 재발급 분기. | 권한 실패 403 handler 는 B3 role 별 endpoint 도입 시 별도 추가. |
| `AuthExceptionHandler` 패키지 범위 = `com.bkurs.auth` | 글로벌 핸들러로 두면 common/* 영향. auth 전용 예외(`InvalidCredentialsException` / `DuplicateEmailException`) 만 다루므로 패키지 한정이 정확. | B2C 의 학원 연결 예외(`AcademyNotFoundException` 등) 는 별도 `academy.web.*ExceptionHandler` 신설. |
| Flyway `V2__create_account.sql` 인덱스 3종 | `uk_email` (login 조회 1건), `idx_status` (운영 화면 status 필터링), `idx_role_status` (role + status 복합 — 운영자가 "활성 강사 목록" 같은 조회 시). | account row 가 만 단위 넘어가면 `idx_role_status` 가 가장 자주 hit. covering index 검토는 100k 이후. |
| `AccountService` 학생 default `PENDING_CONNECT` | 학생은 가입 직후 학원 코드 입력해야 수업 진입 가능 — domain rule. 강사/원장은 학원 측 invite 라 가입 자체가 ACTIVE. | 정책 변경 시 (예: 학원 코드 없이 가입 후 둘러보기 허용) 본 메서드만 수정. |

# 다음 단계 (예정)

- ~~PR 생성 → CI → 머지~~ (진행 중)
- ~~vault 본 task-log status: in-progress → complete~~ (본 commit)
- B2B 시작 — Refresh 토큰 entity + /api/v1/auth/refresh + /logout + CORS

# 회수 / pitfall (누적)

- AuthApiTest 5/5 GREEN 까지 3 단계 디버깅 — Spring Security default 가 인증 실패도 403 으로 내려 401 과 구분 불가. → `HttpStatusEntryPoint(UNAUTHORIZED)` + `AuthExceptionHandler` 조합으로 해결. vault `20-areas/backend/spring/java-spring/pitfalls/spring-security-401-vs-403.md` 신설 후보.

# 참고

- [[../decisions/decision-auth-architecture-jwt-refresh]]
- [[../decisions/decision-account-single-table-role-status]]
- vault [[../../../20-areas/backend/spring/java-spring/common/security-config]] — Spring Security 6 표준 패턴
