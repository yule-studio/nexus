---
title: "B2 Auth 아키텍처 — JWT Access + Refresh 하이브리드"
kind: decision
project: bkurs-be
session_id: bkurs-be-b2-auth
created_at: 2026-05-20T01:00:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - "[[../bkurs-be]]"
  - "[[decision-account-single-table-role-status]]"
  - "[[../task-logs/task-log-b2a-account-jwt]]"
  - "[[../../../20-areas/backend/spring/java-spring/common/security-config]]"
tags: [decision, auth, jwt, security, b2]
---

# 목표

bkurs-be 의 인증 방식 단일 결정 — 4 role (학생/강사/원장/마스터) × 5 status (PENDING_CONNECT / PENDING_APPROVE / ACTIVE / REJECTED / BLOCKED) 워크플로를 안전하게 지지.

# 결정

**JWT Access Token + HttpOnly Refresh Cookie 하이브리드.**

| 토큰 | 방식 | 저장 | 만료 | 갱신 |
|---|---|---|---|---|
| Access | JWT (HS256) | 클라 메모리 또는 localStorage | 30분 | refresh 로 재발급 |
| Refresh | UUID (DB 저장) | HttpOnly + Secure Cookie | 7일 | rotate (사용 시 신규 발급, 옛 row 폐기) |

서버 응답 시:
- Access Token → JSON body `{ accessToken, expiresIn, ... }`
- Refresh Token → Set-Cookie: `Path=/api/v1/auth/refresh; HttpOnly; Secure; SameSite=Strict`

# 대안

| 옵션 | 장점 | 단점 | 결정 |
|---|---|---|---|
| Pure JWT (Access + Refresh JWT, 서버 저장 X) | 완전 stateless, Redis 없이 가능 | **강제 로그아웃 불가** (블랙리스트 만들면 결국 stateful), 비번 변경 후 옛 토큰 살아있음 | ⛔ |
| Pure Session (HttpOnly Cookie + Redis) | XSS 안전, 강제 로그아웃 1줄, 표준 패턴 | 모바일 앱 호환 불편 (쿠키 핸들링), 서버 stateful 필수 | ⛔ |
| **JWT Access + Refresh DB 하이브리드 (선택)** | Access 는 stateless (수평 확장 OK), Refresh 는 강제 폐기 가능 (DB row 삭제), 모바일 / 웹 / SPA 통일 | DB 한 번 더 hit (refresh 시), 구현 복잡도 + | ✅ |
| JWT + Refresh JWT (둘 다 JWT, Refresh 도 stateless) | 가장 stateless | Refresh 폐기 어려움 — 사용자 비번 변경 후 옛 Refresh 가 살아있음 | ⛔ |

# Why (4 구조)

## 왜 필요한가

학원 도메인 특성:
- **학원 측 승인 흐름** (status = PENDING_APPROVE) — 학원이 학생 거절하면 즉시 토큰 무효화 필요
- **학생 졸업 / 강사 이탈 / 학원 계약 해지** — 사후 강제 로그아웃 필수
- **모바일 앱 + 웹 동시 지원** — 동일 토큰 전략으로 코드 일원화

이런 요구가 Pure JWT 의 "stateless 라 폐기 불가" 와 충돌.

## 안 하면 무엇이 깨지는가

Pure JWT 만 쓰면:
- 학원이 학생 거절 → status BLOCKED 로 변경 → 그러나 옛 Access JWT 30분 + Refresh JWT 7일 살아있음
- 학생이 그 사이에 답변 작성 / 다른 학원 코드 입력 등 가능
- 결제 / 회계 영역에서는 더 심각 — 환불 후에도 옛 토큰으로 결제 endpoint 호출 가능

## 대안 비교

위 표. Pure Session 도 충분하나 모바일 앱 (장기적) 호환 위해 JWT 가 안전.

## 트레이드오프

- **Refresh DB hit** — 매 갱신마다 SELECT/UPDATE 1회. 30분 주기라 부담 적음 (Redis 도 검토 가능)
- **CSRF 대응 필요** — HttpOnly Cookie 라 SameSite=Strict + Origin 검증 추가
- **구현 복잡도** — Pure JWT 대비 RefreshToken entity / 갱신 endpoint / 폐기 흐름 추가

# 산출물 (예정)

## B2A (본 PR — issue #3)
- `JwtTokenProvider` — Access JWT 발급/검증 (HS256, 30분, role / accountId claim)
- `JwtAuthenticationFilter` — Bearer 헤더 → SecurityContext
- `SecurityConfig` — permitAll: /api/v1/auth/{signup,login}, /api/v1/health, /actuator/health. 그 외 authenticated

## B2B (다음 — issue 미생성)
- `RefreshToken` entity + `RefreshTokenRepository`
- `POST /api/v1/auth/refresh` — HttpOnly Cookie 의 refresh_token 으로 신규 Access + 신규 Refresh 발급 (rotate)
- `POST /api/v1/auth/logout` — refresh row 삭제 + 쿠키 비움
- CORS 설정 (bkurs-fe origin)

## B2C (그 다음)
- 학원 코드 connect / pending approve / status transition workflow

# 보안 정책 cross-link

- vault [[../../../20-areas/backend/spring/java-spring/common/security-config]] — Spring Security 6 표준 패턴 (CORS / JWT Filter / 401·403 envelope)
- vault [[../../../20-areas/backend/spring/java-spring/common/response-envelope]] — 401/403 응답도 `CommonResponse` 사용

# 참고

- [[../bkurs-be|프로젝트 hub]]
- [[decision-account-single-table-role-status|Account 테이블 전략 결정]]
- [[../task-logs/task-log-b2a-account-jwt|B2A 작업 로그]]
