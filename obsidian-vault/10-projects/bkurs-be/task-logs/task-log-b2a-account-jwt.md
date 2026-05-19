---
title: "B2A — Account entity + JWT 회원가입/로그인 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b2-auth
created_at: 2026-05-20T01:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
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
- C5: 통합 테스트 (signup → login → /me 200 / 잘못된 비번 401 / 만료 토큰 401) + CHANGELOG

# 다음 단계 (예정)

- PR 생성 → CI → 머지
- vault 본 task-log status: in-progress → complete
- B2B 시작 — Refresh 토큰 entity + /api/v1/auth/refresh + /logout + CORS

# 회수 / pitfall (누적 예정)

작업 중 발견 시 본 섹션 + vault `20-areas/backend/spring/java-spring/pitfalls/` 동시 추가.

# 참고

- [[../decisions/decision-auth-architecture-jwt-refresh]]
- [[../decisions/decision-account-single-table-role-status]]
- vault [[../../../20-areas/backend/spring/java-spring/common/security-config]] — Spring Security 6 표준 패턴
