---
title: "B5 결정 — 학생 프로필 (account 1:1) + 학원장 조회 권한"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T10:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, student-profile, school-grade, b5]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-connect-membership-flow]]"
home_hub: bkurs-be
---

# B5 결정 — 학생 프로필 + 조회 권한

## 결론

1. **`student_profile` 별도 테이블, account 와 1:1** (account_id PRIMARY KEY 자체). 학생 외 role 은 row 없음.
2. **학교급은 account.role (STUDENT_HIGH / STUDENT_MIDDLE)** 에서 분기 — profile 에 중복 컬럼 X. 학년 컬럼 `grade` 1-3 (TINYINT).
3. **학생 본인이 작성 / 변경** — upsert 패턴. 학원장은 조회만 (본인 학원 ACTIVE membership 학생 한정).
4. **회원가입 시점에 profile 자동 생성 X** — 학생이 수업 진입 전 자발적으로 채움. bkurs-fe 가 미작성 시 onboarding 화면.

## 왜 필요한가

- bkurs-fe 학생 화면이 학년 / 학교급별 분기 — backend 가 profile 제공 안 하면 화면 분기 불가.
- 학원장이 학생 관리 (학년 / 학교 / 비상연락처) 필요.
- 학원장 / 강사가 수업 / 학사관리 시 학생 정보 일관된 형식으로.

## 안 하면 뭐가 문제

- 학생 정보 (학년 / 학교 / 비상연락처) 없으면 학원 운영 불가.
- account 에 직접 컬럼 추가는 다른 role (강사 / 학원장) 에는 무의미 → 컬럼 NULL 비대화.
- 별도 테이블 분리로 학생 전용 정보 깔끔 + 향후 확장 (학습 스타일 / 관심 분야 등).

## 4 대안 비교 — 데이터 모델

### A. account 에 학생 전용 컬럼 추가 — 기각

- 단점: OWNER/TEACHER/MASTER 는 grade 컬럼 의미 없음 → NULL 만 들어감. 다른 role 정보 추가 시 같은 패턴 반복 → account 비대화.
- **기각**.

### B. student_profile 별도 테이블 (account 1:1) — **채택**

- 장점: 학생 전용 정보 분리. 다른 role 영향 없음. profile 확장 (학습 스타일 / 학교 졸업 등) 시 컬럼 추가만.
- 단점: JOIN 필요. but 학생 정보는 학생/학원장 화면에서만 조회 — N+1 위험 낮음.
- **채택 사유**: 정규화 + 확장성.

### C. user_profile 통합 테이블 (role 무관)

- 단점: 학생/강사/학원장 모두 다른 정보 — 컬럼 polymorphism 어려움 (NULL 폭증).
- **기각**.

### D. JSON 컬럼 (student_profile.metadata)

- 단점: 검증 / 인덱스 / 마이그레이션 어려움.
- **기각**.

## 4 대안 비교 — 학원장 학생 조회 권한

### a. 학원장 본인 학원 ACTIVE membership 학생만 — **채택**

- 장점: ownership 명확. PENDING_APPROVE 학생도 조회 가능해야 승인 결정 가능 (검토 필요).
- 단점: REJECTED 학생은 못 봄.
- **채택 (보강)**: ACTIVE + PENDING_APPROVE 둘 다 허용. REJECTED 는 별도 endpoint.

### b. 학원장 본인 학원 모든 status 학생

- 단점: REJECTED 학생 정보 노출 — 운영 / 감사 목적 외 의미 없음.
- **결정**: 본 PR 은 ACTIVE + PENDING_APPROVE. 향후 admin 별도 endpoint.

### c. 강사 도 본인 학원 학생 조회

- 단점: 강사 권한 매트릭스 명확 후 (B6+) 도입. B5 는 학원장만.
- **후속**: B6+ 에서 강사 권한 확장.

## 테이블 설계

### `student_profile`

```sql
CREATE TABLE student_profile (
  account_id    BIGINT NOT NULL PRIMARY KEY,    -- account 와 1:1
  grade         TINYINT NOT NULL,                -- 1~3
  school_name   VARCHAR(200) NOT NULL,
  birth_date    DATE NULL,
  parent_phone  VARCHAR(32) NULL,
  created_at    DATETIME(6) NOT NULL,
  updated_at    DATETIME(6) NOT NULL,
  CONSTRAINT fk_profile_account FOREIGN KEY (account_id) REFERENCES account(id) ON DELETE CASCADE
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/students/me/profile | STUDENT_* | 본인 — upsert (이미 존재 시 update) |
| GET | /api/v1/students/me/profile | STUDENT_* | 본인 — 미작성 시 404 |
| PATCH | /api/v1/students/me/profile | STUDENT_* | 본인 부분 업데이트 |
| GET | /api/v1/academies/me/students/{accountId}/profile | OWNER | 본인 학원 membership (ACTIVE/PENDING_APPROVE) 학생만 |

## 회귀 라인

`StudentProfileApiTest`:
1. STUDENT POST → 201 (신규)
2. STUDENT POST (이미 존재) → 200 + upsert
3. STUDENT GET → 200
4. STUDENT PATCH → 부분 변경
5. 학원장 role POST → 403
6. 미작성 GET → 404
7. OWNER 본인 학원 학생 조회 → 200
8. OWNER 다른 학원 학생 → 403
9. OWNER 본인 학원 미작성 학생 → 404 (학생은 있지만 profile 없음)
10. STUDENT_MIDDLE 동일 흐름 → 정상 (학교급은 account.role 에서 분기)

## 관련

- [[decision-academy-connect-membership-flow]] — 학원장 ownership 검증 (membership 학생만)
- 후속: B6+ 강사 권한 매트릭스 (강사도 본인 학원 학생 조회 가능 검토)
