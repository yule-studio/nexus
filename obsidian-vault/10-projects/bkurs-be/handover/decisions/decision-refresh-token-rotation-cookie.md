---
title: "B2B 결정 — Refresh 토큰 회전 + HttpOnly Cookie 전달"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T04:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, auth, refresh-token, cookie, security]
related:
  - "[[../bkurs-be]]"
  - "[[decision-auth-architecture-jwt-refresh]]"
  - "[[decision-account-single-table-role-status]]"
  - "[[../task-logs/task-log-b2a-account-jwt]]"
home_hub: bkurs-be
---

# B2B 결정 — Refresh 토큰 회전 + HttpOnly Cookie 전달

## 결론

1. **Refresh 토큰은 DB 테이블(`refresh_token`)에 저장** — token 자체는 클라이언트만 보관, 서버는 hash 만.
2. **rotation 정책**: refresh 사용 시마다 새 refresh 발급 + 직전 token revoke. 동시에 사용 시 reuse detection 으로 전체 family revoke.
3. **HttpOnly Cookie 전송** (path=`/api/v1/auth`, Secure=prod-only, SameSite=Lax). access 는 JSON body 로 유지 (Authorization 헤더 사용).
4. **만료**: refresh 7일, rotation 시 갱신. access 는 B2A 결정 그대로 30분.
5. **logout** 은 cookie 의 refresh 를 받아 DB 에서 revoke + Set-Cookie 로 만료 cookie 전송.

## 왜 필요한가

- B2A 의 access only (30분) 만으로는 30분마다 재로그인 → bkurs-fe UX (3시간 학습 세션) 와 안 맞음.
- 모바일/PWA 사용자도 같은 origin 으로 접근 → cookie 전달이 단순.
- 학원·강사·결제 데이터를 다루는 시스템이라 token 탈취 시 빠른 revoke 가 필수.

## 안 하면 뭐가 문제

- localStorage refresh 보관 시 XSS 1번에 토큰 통째 유출 → 학원 전체 운영 데이터 노출 가능.
- DB 저장 없으면 logout 이 "클라 토큰 삭제" 뿐 — 서버는 만료까지 모름 → token 탈취자가 계속 사용.
- rotation 없으면 한 번 유출된 refresh 가 7일간 계속 access 발급기로 작동.

## 4 대안 비교

### A. Refresh 토큰 없이 access only (현 B2A) — 기각

- 장점: 단순. 서버 stateless 유지.
- 단점: 30분마다 재로그인. bkurs 사용 시나리오(긴 학습 세션)와 충돌.
- **기각 사유**: UX 자체가 깨짐.

### B. JWT refresh (DB 저장 없음) — 기각

- 장점: stateless 유지.
- 단점: revoke 불가능. logout 이 클라 측 cookie 삭제만. token 탈취 시 무력.
- **기각 사유**: 학원/결제 시스템이라 revoke 능력 필수.

### C. DB refresh, rotation 없음 (single long-lived token) — 기각

- 장점: 구현 단순. refresh 한 번 발급하고 7일간 그대로.
- 단점: 탈취된 refresh 가 7일간 살아있고, 정상 클라와 공존 가능 (탐지 불가).
- **기각 사유**: rotation 없는 long-lived token 은 OWASP 권고 위반.

### D. DB refresh + rotation + reuse detection — **채택**

- 장점:
  - revoke 가능 (DB row 만 마킹).
  - rotation 으로 탈취 윈도우 단축 (1회용).
  - 동시 사용 감지 → family revoke (탈취자 사용 즉시 정상 클라도 함께 logout, 사용자가 재로그인하면서 인지).
- 단점: DB 쓰기 추가 (refresh 1회당 1 row insert + 직전 row update). HikariCP 풀 영향 미미.
- **채택 사유**: OWASP "Token Storage Cheat Sheet" + Auth0 / Okta 권고 형식 그대로.

## 테이블 설계 — `refresh_token`

```sql
CREATE TABLE refresh_token (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  account_id      BIGINT NOT NULL,
  token_hash      VARCHAR(255) NOT NULL,         -- SHA-256(token), token 평문은 저장 안 함
  family_id       VARCHAR(64) NOT NULL,           -- rotation 추적용 UUID
  parent_id       BIGINT NULL,                    -- 회전 직전 row id (chain 추적)
  user_agent      VARCHAR(500) NULL,              -- 발급 시 User-Agent (감사용)
  ip              VARCHAR(45) NULL,               -- IPv4/IPv6
  issued_at       DATETIME(6) NOT NULL,
  expires_at      DATETIME(6) NOT NULL,
  revoked_at      DATETIME(6) NULL,
  revoke_reason   VARCHAR(50) NULL,               -- 'rotation' / 'logout' / 'reuse_detected' / 'admin'
  CONSTRAINT fk_refresh_account FOREIGN KEY (account_id) REFERENCES account(id) ON DELETE CASCADE,
  UNIQUE KEY uk_token_hash (token_hash),
  KEY idx_account_active (account_id, revoked_at),
  KEY idx_family (family_id),
  KEY idx_expires_at (expires_at)
);
```

- `token_hash` UNIQUE → reuse detection 의 핵심 (같은 token 으로 2회 요청 = 즉시 family revoke).
- `idx_account_active` → "이 계정의 active refresh 목록" 운영 화면용.
- `idx_expires_at` → 만료 cleanup job 의 인덱스 (B6+ scheduler).

## Cookie 정책

| 속성 | 값 | 이유 |
|---|---|---|
| `HttpOnly` | true | XSS 차단 — JS 가 cookie 못 읽음 |
| `Secure` | prod 만 true | local dev 의 http 환경 호환. application-prod.yml 의 별도 설정 |
| `SameSite` | `Lax` | CSRF 1차 방어. bkurs-fe 가 같은 site 면 충분. cross-site iframe 시 `None` 으로 조정 |
| `Path` | `/api/v1/auth` | refresh / logout endpoint 에만 cookie 보내짐 (다른 endpoint 의 요청 헤더 비대화 방지) |
| `Max-Age` | 7일 (604800초) | refresh 만료와 동일 |
| `Name` | `bkurs_rt` | 직관적 + 충돌 방지 prefix |

## CORS 정책 (별도 결정)

- `cors.allowed-origins`: application-local.yml 에 `http://localhost:5173` (bkurs-fe vite), prod 는 `https://app.bkurs.kr` (예정).
- `cors.allowed-methods`: GET, POST, PUT, DELETE, PATCH, OPTIONS.
- `cors.allowed-headers`: Authorization, Content-Type.
- `cors.allow-credentials`: **true** — cookie 송수신 필수.
- `cors.exposed-headers`: 없음 (B2B 단계 기준).

## 회귀 라인

`AuthApiTest` 확장 + 신규 `RefreshTokenApiTest`:

1. login → access + Set-Cookie(refresh) 둘 다 응답에 있다.
2. cookie 와 함께 `POST /api/v1/auth/refresh` → 새 access + 새 Set-Cookie. 직전 token DB 에서 `revoked_at` 채워짐.
3. 직전 token 으로 refresh 재시도 → 401 + family 전체 revoke.
4. cookie 없이 refresh → 401.
5. logout → cookie 의 refresh DB revoked + 응답 Set-Cookie 가 빈 값 + Max-Age 0.
6. 만료된 refresh 로 refresh → 401.

## 관련

- [[decision-auth-architecture-jwt-refresh]] — 상위 결정 (refresh 도입 자체)
- [[../bkurs-be]] — PR 시리즈 위치 (B2B)
- OWASP Token Storage / Refresh Rotation cheat sheet
- vault [[../../../20-areas/backend/spring/java-spring/pitfalls/spring-security-401-vs-403]] — 401 의미 일치
