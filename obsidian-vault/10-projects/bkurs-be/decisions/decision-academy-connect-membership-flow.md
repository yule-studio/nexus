---
title: "B2C 결정 — 학원 연결 / 승인 / 상태 전이 (membership 별도 테이블)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T05:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, academy, membership, status-transition, b2c]
related:
  - "[[../bkurs-be]]"
  - "[[decision-account-single-table-role-status]]"
home_hub: bkurs-be
---

# B2C 결정 — 학원 연결 / 승인 / 상태 전이 (membership 별도 테이블)

## 결론

1. **`student_academy_membership` 별도 테이블** (account FK + academy FK + status). account 테이블에 academy_id 컬럼 X.
2. **AcademyCode** 는 학원 한 곳당 단일 active code — code 자체로 academy 식별 + 사용 가능 여부 검증.
3. **상태 전이**: account.status (`PENDING_CONNECT → PENDING_APPROVE → ACTIVE`) + membership.status (`PENDING_APPROVE → ACTIVE / REJECTED`) **이중 트랙**. account.status 는 "로그인 후 어느 화면 보낼지", membership.status 는 "이 학원과의 관계 상태".
4. **권한**: connect 는 학생만 (PENDING_CONNECT 상태에서만). approve/reject 는 학원장 (OWNER) — `@PreAuthorize("hasRole('OWNER')")`.

## 왜 필요한가

- 학생은 가입 → 학원 코드 입력 → 학원 승인 후 수업 진입 (bkurs-fe 의 4 단계 flow 그대로).
- 한 학생이 미래에 학원 이동 / 복수 학원 동시 (보충 학원) 시나리오 발생 가능 — membership 분리하면 N:M 으로 확장 가능.
- 운영자가 "이 학원의 승인 대기 학생" / "이 학생의 학원 이력" 양방향 조회 필요.

## 안 하면 뭐가 문제

- account.academy_id 단일 컬럼이면 학원 이동 시 history 소실. "이 학원 출신 학생 명부" 같은 정산/감사 query 불가.
- 승인 거부 (REJECTED) 도 데이터로 보존 안 됨 → 같은 코드 재시도 spam 방지 못함.
- account.status 만으로 학원 관계 표현하면, 학원 입장의 "승인 대기 큐" 가 account 테이블 전체를 스캔해야 함 (학원 컬럼 인덱스 비효율).

## 4 대안 비교

### A. account.academy_id 단일 컬럼 (membership 없음) — 기각

- 장점: 단순. 1 query.
- 단점: 학원 이동 history 없음. N:M 불가. 정산/감사용 학원별 학생 명부 인덱스 비효율.
- **기각 사유**: bkurs 운영 시나리오 (이력 관리 / 멀티 학원) 와 충돌.

### B. account 의 status 만 사용, 학원 관계는 별도 cache — 기각

- 단점: cache 가 SSoT 와 분리 — staleness / 동기화 버그 위험. 학원장 화면이 cache 보면 승인 누락 가능.
- **기각 사유**: 인증/권한 같은 critical path 가 cache 의존하면 안 됨.

### C. membership 테이블 + status 컬럼만 (account.status 그대로 PENDING_CONNECT) — 기각

- 단점: 학생이 로그인 후 화면 분기 (Connect 화면 / Pending 화면 / Active 화면) 가 N:M membership query 가 필요. JWT 의 status 클레임 활용 못함.
- 운영자도 account 만 보고는 "이 학생 어디까지 진행됨" 즉시 못 봄.
- **기각 사유**: UX bandwith (every login → 추가 query) 부담.

### D. membership 별도 테이블 + account.status 이중 트랙 — **채택**

- 장점:
  - account.status 가 "사용자 관점" (JWT 클레임 그대로 화면 분기 가능).
  - membership.status 가 "학원 관점" (학원장이 학원별 큐 즉시 조회).
  - 학원 이동 시 새 membership row insert + 직전 ACTIVE 를 INACTIVE 마킹 (B3+) — history 보존.
  - REJECTED 도 보존 → 같은 코드 spam 방지 / 거부 사유 audit.
- 단점: signup 직후 connect 시 2 row 쓰기 (account.status update + membership insert) + transaction 묶음.
- **채택 사유**: bkurs 의 운영 시나리오 (이력 + 학원장 큐 + UX 화면 분기) 모두 지원하는 유일 형태.

## 테이블 설계

### `academy`

```sql
CREATE TABLE academy (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  name        VARCHAR(200) NOT NULL,
  owner_account_id BIGINT NOT NULL,        -- 학원장 계정 (단일, B2C 기준)
  created_at  DATETIME(6) NOT NULL,
  updated_at  DATETIME(6) NOT NULL,
  CONSTRAINT fk_academy_owner FOREIGN KEY (owner_account_id) REFERENCES account(id),
  KEY idx_academy_owner (owner_account_id)
);
```

### `academy_code`

```sql
CREATE TABLE academy_code (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  academy_id   BIGINT NOT NULL,
  code         VARCHAR(32) NOT NULL,
  active       TINYINT(1) NOT NULL DEFAULT 1,    -- 학원장이 비활성화 가능 (B3 에서 expires_at / max_uses)
  created_at   DATETIME(6) NOT NULL,
  CONSTRAINT fk_code_academy FOREIGN KEY (academy_id) REFERENCES academy(id) ON DELETE CASCADE,
  UNIQUE KEY uk_code (code),                     -- 전역 unique
  KEY idx_code_academy_active (academy_id, active)
);
```

### `student_academy_membership`

```sql
CREATE TABLE student_academy_membership (
  id             BIGINT AUTO_INCREMENT PRIMARY KEY,
  account_id     BIGINT NOT NULL,
  academy_id     BIGINT NOT NULL,
  status         VARCHAR(32) NOT NULL,           -- PENDING_APPROVE / ACTIVE / REJECTED / INACTIVE
  requested_at   DATETIME(6) NOT NULL,
  decided_at     DATETIME(6) NULL,
  decided_by_account_id BIGINT NULL,             -- 승인/거부한 학원장 / 강사
  reject_reason  VARCHAR(500) NULL,
  CONSTRAINT fk_membership_account FOREIGN KEY (account_id) REFERENCES account(id),
  CONSTRAINT fk_membership_academy FOREIGN KEY (academy_id) REFERENCES academy(id),
  UNIQUE KEY uk_membership_active (account_id, academy_id, status),  -- 같은 학원 중복 신청 방지
  KEY idx_membership_academy_status (academy_id, status),            -- 학원장 큐
  KEY idx_membership_account (account_id)
);
```

## 상태 전이 다이어그램

```
account.status:
  PENDING_CONNECT --(connect by code)--> PENDING_APPROVE --(owner approve)--> ACTIVE
                                                         \--(owner reject)--> REJECTED (account)

membership.status:
  (none) --(connect)--> PENDING_APPROVE --(approve)--> ACTIVE
                                        \--(reject)---> REJECTED
```

- 학생 입장: account.status 가 화면 분기 키 (JWT 클레임).
- 학원장 입장: `membership.status='PENDING_APPROVE'` 큐 조회.

## 권한 정책

| endpoint | role | 추가 조건 |
|---|---|---|
| POST /api/v1/auth/connect | STUDENT_HIGH / STUDENT_MIDDLE | account.status = PENDING_CONNECT |
| POST /api/v1/academies/memberships/{accountId}/approve | OWNER | 본인 소유 학원의 membership 만 |
| POST /api/v1/academies/memberships/{accountId}/reject | OWNER | 본인 소유 학원의 membership 만 |
| GET /api/v1/academies/me/memberships/pending | OWNER | 본인 소유 학원 |

`@PreAuthorize("hasRole('OWNER')")` 로 role 체크 + service layer 에서 ownership 검증 (다른 학원 학생 승인 차단).

## 회귀 라인

`ConnectApprovalApiTest`:

1. student PENDING_CONNECT + 올바른 code → connect → membership PENDING_APPROVE + account.status PENDING_APPROVE.
2. 잘못된 code → 404 (또는 400, body 메시지 불일치 방지).
3. 비활성 code → 409 (`code_inactive`).
4. 이미 PENDING_APPROVE 인 학생이 다시 connect → 409 (`already_pending`).
5. 학원장(OWNER) 가 본인 학원의 pending 학생 approve → membership ACTIVE + account.status ACTIVE.
6. 학원장 reject → membership REJECTED + account.status REJECTED.
7. 다른 학원장(또는 비-OWNER) 이 approve 시도 → 403.
8. 이미 ACTIVE 인 membership 재approve → 409 (idempotent vs error 정책 결정 필요 — 본 결정에서는 409).

## 관련

- [[decision-account-single-table-role-status]] — account.status / Role enum
- [[../bkurs-be]] — PR 시리즈 위치 (B2C)
