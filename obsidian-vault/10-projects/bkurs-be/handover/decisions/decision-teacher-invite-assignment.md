---
title: "B3B 결정 — 강사 학원 소속 (invite 기반 + 별도 assignment 테이블)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T07:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, teacher, invite, assignment, b3b]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-management-owner-policy]]"
  - "[[decision-academy-connect-membership-flow]]"
home_hub: bkurs-be
---

# B3B 결정 — 강사 학원 소속 (invite 기반 + 별도 assignment 테이블)

## 결론

1. **invite 기반 가입** — 학원장이 invite token 생성 → 강사가 일반 signup 후 token 으로 학원 연결.
2. **별도 `teacher_academy_assignment` 테이블** — 학생 membership 과 별개. 강사는 PENDING_APPROVE flow 없이 invite → 즉시 ACTIVE.
3. **invite 만료 7일 + 1회용** — 보안 + 운영 단순.
4. **권한**: 학원장은 본인 학원의 invite/assignment 만 조작. 강사는 본인 assignment 만 조회.

## 왜 필요한가

- 강사도 학원에 속해야 향후 (B3C 학원 공지, B4+ 학원 정보 변경 권한, 수업 등록 등) 권한 분기 가능.
- 학생과 달리 강사는 학원장이 신원 확인 후 초대 — invite 흐름이 자연스러움 (학원장이 강사 이메일에 직접 알려주거나 운영 시스템에서 발송).
- PENDING_APPROVE 단계 불필요 — 학원장이 직접 invite 했으니 즉시 ACTIVE.

## 안 하면 뭐가 문제

- 강사 가입 endpoint 없으면 학원장이 SQL 로 강사 추가 → 운영 위험.
- 학생 membership 과 같은 테이블 쓰면 PENDING_APPROVE 의미 충돌 + 학원장 큐가 학생/강사 섞임 → 운영 화면 복잡.
- invite 만료 없으면 한 번 발급된 token 영구 유효 → 보안 사고 가능.

## 4 대안 비교

### A. 학원장이 강사 계정 직접 생성 (이메일 + 임시 비번) — 기각

- 단점: 임시 비번 전달 보안 위험 (이메일 평문, 운영자 노출). 강사가 비번 변경 안 하면 학원장이 강사 계정 접근 가능.
- **기각 사유**: 자격 분리 원칙 위반.

### B. invite token + 강사 일반 signup → token 으로 학원 연결 — **채택**

- 장점: 강사 본인이 비번 설정. invite token 은 학원-연결 권한만, 계정 자격은 강사 본인. 학원장 운영 부담 최소.
- 단점: 가입 + connect 2 단계 — UX 약간 복잡.
- **채택 사유**: 보안 + UX + 운영 부담 모두 균형.

### C. signup-with-invite 통합 (한 endpoint) — 미래 후보

- 장점: 1 단계 UX.
- 단점: B2A signup endpoint 와 분기 — DTO / validation / 응답 형식 모두 분기.
- **결정**: B3B 는 B 채택, signup-with-invite 는 운영 피드백 후 별도 PR 검토.

### D. 학원장이 강사 이메일 입력 → 시스템이 신규 가입 링크 발송 — 기각

- 단점: 이메일 발송 인프라 (SES / Mailgun 등) 필요. B3B 범위 초과.
- **기각 사유**: 인프라 의존. invite token 만 API 응답에 포함하고 학원장이 강사에게 직접 전달 → 이메일 인프라 도입 후 자동화.

## 4 대안 비교 — 데이터 모델

### a. `student_academy_membership` 에 role 컬럼 추가

- 단점: 학생 membership 의 status (PENDING_APPROVE/...) 와 강사 (ACTIVE/REMOVED) 의미 충돌. 학원장 큐 query 가 role 필터 필수.
- **기각**.

### b. `academy_member` 통합 테이블 (모든 학원 관계) — 미래 후보

- 장점: 학원 ↔ 사용자 N:M 단일 모델.
- 단점: B3B 범위에서 B2C 의 student_academy_membership 마이그레이션 필요 → 데이터 모델 큰 변화.
- **결정**: B4 학원 이관 / 분원 시 통합 검토.

### c. 별도 `teacher_academy_assignment` 테이블 — **채택**

- 장점: 학생 / 강사 의미 분리. 학원장 운영 화면 query 단순.
- 단점: B4+ 통합 시 마이그레이션 필요.
- **채택 사유**: 단계적 도입.

## 테이블 설계

### `teacher_invite`

```sql
CREATE TABLE teacher_invite (
  id                BIGINT AUTO_INCREMENT PRIMARY KEY,
  academy_id        BIGINT NOT NULL,
  token             VARCHAR(64) NOT NULL,        -- 32-byte SecureRandom base64url
  issued_by_owner_id BIGINT NOT NULL,
  issued_at         DATETIME(6) NOT NULL,
  expires_at        DATETIME(6) NOT NULL,
  used_at           DATETIME(6) NULL,
  used_by_account_id BIGINT NULL,
  CONSTRAINT uk_teacher_invite_token UNIQUE (token),
  KEY idx_invite_academy (academy_id),
  CONSTRAINT fk_invite_academy FOREIGN KEY (academy_id) REFERENCES academy(id) ON DELETE CASCADE
);
```

### `teacher_academy_assignment`

```sql
CREATE TABLE teacher_academy_assignment (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  account_id  BIGINT NOT NULL,
  academy_id  BIGINT NOT NULL,
  status      VARCHAR(32) NOT NULL,           -- ACTIVE / REMOVED
  joined_at   DATETIME(6) NOT NULL,
  removed_at  DATETIME(6) NULL,
  removed_by_owner_id BIGINT NULL,
  CONSTRAINT fk_assignment_account FOREIGN KEY (account_id) REFERENCES account(id),
  CONSTRAINT fk_assignment_academy FOREIGN KEY (academy_id) REFERENCES academy(id),
  KEY idx_assignment_academy_status (academy_id, status),
  KEY idx_assignment_account_status (account_id, status)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/teacher-invites | OWNER | 본인 학원 |
| GET | /api/v1/academies/me/teacher-invites | OWNER | 본인 학원 |
| POST | /api/v1/auth/teacher-connect | TEACHER | invite valid + 미사용 + 미만료 |
| GET | /api/v1/teachers/me/academy | TEACHER | 본인 ACTIVE assignment |
| GET | /api/v1/academies/me/teachers | OWNER | 본인 학원 |
| POST | /api/v1/academies/me/teachers/{assignmentId}/remove | OWNER | 본인 학원 assignment |

## 회귀 라인

`TeacherInviteApiTest`:
1. OWNER invite 발급 → 201 + token 응답
2. TEACHER signup → connect with token → assignment ACTIVE
3. 잘못된 token → 404
4. 만료된 token → 409
5. used token 재사용 → 409
6. OWNER 강사 목록 조회 → 1 row
7. OWNER 강사 remove → status REMOVED
8. STUDENT 가 teacher-connect 시도 → 403
9. 다른 학원장이 우리 강사 remove → 403
10. removed 강사가 본인 학원 조회 → 404

## 관련

- [[decision-academy-management-owner-policy]] — 본 결정의 상위 (학원장 1인 1학원 / OWNER 권한)
- [[decision-academy-connect-membership-flow]] — 학생 membership 흐름 (비교 대상)
