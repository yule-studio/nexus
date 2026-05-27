---
title: "B5 — 학생 프로필 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b5-student
created_at: 2026-05-20T10:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-student-profile-school-grade]]"
  - "[[task-log-b4d-owner-transfer]]"
tags: [task-log, student-profile, b5]
---

# 작업

GitHub issue [#23](https://github.com/seum-running/bkurs-be/issues/23) — 학생 프로필 + 학원장 조회.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 데이터 모델 | student_profile 별도 테이블 (account 1:1) | [[../decisions/decision-student-profile-school-grade]] |
| 학교급 | account.role (STUDENT_HIGH / STUDENT_MIDDLE) 에서 분기 | profile 에 중복 컬럼 X |
| grade | INT 1-3 (TINYINT 충돌로 변경) | application 검증 |
| 학원장 조회 | ACTIVE + PENDING_APPROVE membership 학생만 | 승인 결정에 필요한 정보 |
| profile 자동 생성 X | 학생이 자발 작성, bkurs-fe onboarding 화면 | |

# 진행

## 2026-05-20

### 1. branch / issue
- B4D 머지 대기 중 stacked PR — base=feature/issue-21-b4d-owner-transfer
- issue #23 + decision 노트 (4 대안 × 2 — 데이터 모델 + 조회 권한)

### 2. 코드 (commit 단위)
- C1: StudentProfile entity + V11 (account 1:1, grade TINYINT initial)
- C2: 3 예외 + StudentProfileService (upsert/patch/get + 학원장 ownership 검증)
- C3: 3 DTO + 2 controller (학생 / 학원장) + ExceptionHandler
- C4: StudentProfileApiTest 10 + V11 grade TINYINT → INT 수정 (JPA Integer 와 schema validate 충돌) + CHANGELOG B5

### 3. PR
- PR [#24](https://github.com/seum-running/bkurs-be/pull/24) (feature/issue-23 → feature/issue-21-b4d-owner-transfer, stacked)
- 회귀 **102/102 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| student_profile 별도 테이블 (account 1:1) | account 에 직접 컬럼 추가는 OWNER/TEACHER/MASTER 에 무의미 (NULL 비대화). 별도 분리로 학생 전용 확장 (학습 스타일 / 학교 졸업 등). | user_profile 통합은 polymorphism 어려움. JSON 컬럼은 검증/인덱스 어려움. |
| account_id PRIMARY KEY (1:1 강제) | 별도 FK 컬럼 + unique constraint 두는 것보다 schema 가 1:1 명확. | account 1:N 학생 시 schema 변경 필요하지만 도메인상 불가. |
| 학교급 = account.role (profile 컬럼 X) | 같은 정보 두 곳 저장 (account.role / profile.schoolLevel) = 동기화 위험. role 에서 분기. | role 변경 (중→고 진학) 시 별도 endpoint (B6+ 진학 처리). |
| grade INT (TINYINT 회수) | Hibernate Integer 가 schema validate 시 TINYINT 와 충돌 → INT 로. 4 bytes 차이 무시 가능 (학생 수 만 단위). | TINYINT 유지하려면 entity Short / Byte 사용 또는 columnDefinition 명시. |
| upsert 패턴 (POST = 신규 OR update) | 학생이 첫 작성 / 재작성 같은 UX — 별도 endpoint 분리 부담. 멱등 보장. | PUT vs POST 의 의미상 PUT 이 더 정확하지만 REST 엄격 적용 시 클라 부담. |
| PATCH null 무시 | 학생이 일부만 변경 (학년만 / school 만) — 클라가 변경 안 한 필드를 null 전송. entity update 가 null 무시. | 명시적 "지우기" 는 별도 endpoint (B6+). |
| 학원장 조회 = ACTIVE + PENDING_APPROVE | PENDING_APPROVE 학생도 학원장이 봐야 승인 결정 (학년 등 정보 필요). REJECTED 는 별도 admin. | 강사 학생 조회는 B6+ 강사 권한 매트릭스. |
| 학원장 controller 클래스 레벨 @PreAuthorize OWNER | path 가 /academies/me/students/* 라 OWNER 전용 명확. | 학생 controller 는 인증 사용자 누구나 진입 → service role 검증. |
| 학생 controller 의 role 검증 service 위임 | controller 에 @PreAuthorize 두면 학원장/강사 진입 시 default 403 (의미 모호). service 의 NotStudentProfileOwner 가 명확한 "students_only" 응답. | 향후 강사 학생 조회 추가 시 controller 다중 role 처리 어려움 — service 분리가 유연. |
| 미작성 학생 조회 404 (PATCH 도) | PATCH 가 신규 생성도 하면 UX 모호 (POST 와 차이 사라짐). 명시적 분리. | upsert 의도면 POST 사용. |
| StudentProfileExceptionHandler 도메인 한정 (com.bkurs.student) | 다른 도메인 (academy / plan / announcement) 의 동일 status code 매핑과 envelope 일관성 유지. | global handler 통합은 B5 끝나고 별도 cleanup PR. |
| OwnerStudentProfileController 분리 (StudentProfileController 와 별도) | path prefix 다름 (/me/profile vs /me/students/{id}/profile). 같은 controller 면 라우팅 모호 + 책임 모호. | 향후 강사 학생 조회 (B6+) 추가 시 또 다른 controller. |

# 다음 단계 (예정)

- B2A → ... → B5 순으로 사용자 manual merge → CI 자동 실행
- B6 시작 — 수업 / 일정 / 학사 (강사 권한 매트릭스 포함)
- B7+ — 학습 콘텐츠 (생기부 / 자소서 / 예상질문 / 기출 ...)

# 회수 / pitfall (누적)

- **grade TINYINT vs INT 함정** — Hibernate 의 Java Integer 매핑은 schema validate 시 INT 만 허용. TINYINT 는 entity 가 Short / Byte 또는 columnDefinition 명시 필요. application 검증으로 충분하면 INT 가 단순.
- vault 함정 노트 후보 (Hibernate schema validate vs DB type 매핑 일반).

# 참고

- [[../decisions/decision-student-profile-school-grade]]
- [[task-log-b4d-owner-transfer]]
