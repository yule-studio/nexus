---
title: "B6C — 강사 권한 매트릭스 정밀화 (B6 완결) 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b6-course
created_at: 2026-05-20T12:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-teacher-permission-matrix]]"
  - "[[task-log-b6b-session-attendance]]"
tags: [task-log, teacher-permission, b6c]
---

# 작업

GitHub issue [#29](https://github.com/seum-running/bkurs-be/issues/29) — 강사 권한 매트릭스 정밀화.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 강사 학생 조회 | 본인 배정 수업 ACTIVE 수강생만 | [[../decisions/decision-teacher-permission-matrix]] |
| 강사 수업 수정 | description 만 (name/max/teacher 는 OWNER) | 일상 운영 자율 / 정책 보존 |
| OWNER 출결 path | 별도 controller (TEACHER path 와 분리) | 권한 명확 |
| 강사 호출 path | /teachers/me/* (TEACHER 전용 @PreAuthorize) | role 명확 |

# 진행

## 2026-05-20

### 1. branch / issue
- B6B 머지 대기 중 stacked PR — base=feature/issue-27-b6b-session-attendance
- issue #29 + decision 노트

### 2. 코드 (commit 단위)
- C1: Course.updateLimited + StudentProfileService.getStudentOnly (service 헬퍼)
- C2: 1 DTO + 3 controller (OwnerAttendance / TeacherCourseUpdate / TeacherStudentProfile)
- C3: CourseService.updateByTeacher write 트랜잭션 + TeacherPermissionApiTest 10 + CHANGELOG B6C
- C4: CHANGELOG fixup (직전 누락분)

### 3. PR
- PR [#30](https://github.com/seum-running/bkurs-be/pull/30) (feature/issue-29 → feature/issue-27-b6b-session-attendance, stacked)
- 회귀 **132/132 GREEN** — **B6 시리즈 완결**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| 강사 description 만 수정 | name/maxStudents/teacherAssignmentId = 학원 정책 (가격 / 정원 / 배정). 학원장 보존. description = 강사 일상 운영. | 향후 운영 피드백 후 일부 확장 (room/notes 등). |
| TeacherUpdateCourseRequest DTO 신규 (다른 필드 X) | request 에 다른 필드 없음 = controller 가 받지도 못함. name/max 변경 시도 무시 자연스러움. | UpdateCourseRequest 재사용은 의미 모호 (DTO 가 권한 신호 역할). |
| OwnerAttendanceController 별도 | TEACHER controller path 에 OWNER 허용은 권한 모호. 별도 path = 명확. | 메서드별 @PreAuthorize (TEACHER OR OWNER) 도 가능, 명확성 우선. |
| TeacherStudentProfileController 컨트롤러가 ownership 검증 | 강사 ↔ 학생 ownership 은 강사 → course → enrollment chain. service 가 받기 어색. controller 가 cross-domain 검증. | StudentProfileService.getByTeacher 시그니처 도입은 service 가 강사 도메인 의존 — 패키지 cross. controller 분기가 깔끔. |
| StudentProfileService.getStudentOnly 신규 메서드 | controller 가 ownership 검증 후 단순 fetch. service 는 role 검증만. | service 가 본인학원 검증 강제하면 강사 path 에서 학원 추적 어려움. |
| CourseService.updateByTeacher (write 트랜잭션) | controller 직접 entity mutation = readOnly 컨텍스트 = flush 안 됨. service 의 write 트랜잭션으로 managed entity flush 보장. | service 가 entity 직접 fetch + update = JPA 1-level cache + dirty checking. |
| 본인 배정 검증 (listByTeacher + filter) | 강사 본인 ACTIVE assignment 의 수업 목록 → courseId 매칭. service 가 ownership 검증 응집. | direct query (course.teacher_assignment_id = ?) 도 가능. listByTeacher 가 단일 진입점. |
| TeacherPermissionApiTest 의 8번 (다른 학원 학생) | 학원 격리 명시 검증 — 다른 학원 강사가 다른 학원 학생 못 봄. | enrollment N+1 으로 검증하지만 학원 50개 × 수업 100개 가정 시 service query 분리. |
| OWNER+TEACHER recorded_by 구분 회귀 | upsert update 시 recorded_by 가 가장 최근 actor 로 갱신 — audit 의미. | append-only (별도 audit row) 도 가능. 단순 갱신이 운영 단순. |
| description PATCH 의 entity flush 회수 | readOnly 트랜잭션 — controller 가 detached entity mutation = flush 안 됨. service 위임 + write 트랜잭션. | vault 함정 노트 후보 (entity mutation outside write transaction). |

# 다음 단계 (예정)

- B2A → ... → B6C 순으로 사용자 manual merge → CI 자동 실행
- B7 시작 — 학습 콘텐츠 baseline (생기부 / 자소서 / 예상질문 / 기출 / 모의면접 / 시뮬레이션 / 토론 / 수행평가 / 독서 / 숙제 / 학사관리)
- B7 은 큰 시리즈 — 도메인별 분할 (B7A 생기부, B7B 자소서 ...) 검토

# 회수 / pitfall (누적)

- **entity mutation outside write transaction** — controller 가 listByTeacher (readOnly) 의 결과 entity 를 mutation 해도 flush 안 됨. service 의 write 트랜잭션으로 위임 필수. vault 함정 노트 후보.

# 참고

- [[../decisions/decision-teacher-permission-matrix]]
- [[task-log-b6b-session-attendance]]
