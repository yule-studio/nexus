---
title: "B6B — CourseSession + Attendance (시간표 + 출결) 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b6-course
created_at: 2026-05-20T11:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-session-attendance-model]]"
  - "[[task-log-b6a-course-enrollment]]"
tags: [task-log, session, attendance, b6b]
---

# 작업

GitHub issue [#27](https://github.com/seum-running/bkurs-be/issues/27) — CourseSession + Attendance.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 모델 | course_session + attendance 별도 (정규화) | [[../decisions/decision-session-attendance-model]] |
| status enum | PRESENT / ABSENT / LATE / EXCUSED (4종) | application 검증 |
| upsert | (session, account) UNIQUE → 같은 학생 같은 세션 update | 멱등 |
| 입력 권한 | TEACHER (본인 배정) + OWNER (강사 부재 대체) | recorded_by audit |
| 시간 invariant | end > start (entity 강제) | IllegalArgument → 400 |

# 진행

## 2026-05-20

### 1. branch / issue
- B6A 머지 대기 중 stacked PR — base=feature/issue-25-b6a-course-enrollment
- issue #27 + decision 노트

### 2. 코드 (commit 단위)
- C1: CourseSession + Attendance entity + V13 + AttendanceStatus enum + invariant + Repository
- C2: 2 예외 + SessionService (OWNER CRUD) + AttendanceService (upsert + role 분기)
- C3: 5 DTO + 3 controller (Owner/Teacher/Student) + ExceptionHandler 매핑 + IllegalArgumentException 400
- C4: SessionAttendanceApiTest 10 + 10 기존 테스트 cleanup 보강 (V13 FK course.teacher_assignment_id) + CHANGELOG B6B

### 3. PR
- PR [#28](https://github.com/seum-running/bkurs-be/pull/28) (feature/issue-27 → feature/issue-25-b6a-course-enrollment, stacked)
- 회귀 **122/122 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| course_session + attendance 정규화 | JSON 컬럼 안티패턴 회피. 인덱스 활용 (course/session/account 양방향). | row 비대 (학원/년 150k) 인덱스로 해결. |
| AttendanceStatus 4종 | PRESENT/ABSENT/LATE/EXCUSED — 학원 운영 표준. | 향후 결시(SICK) / 외출(OUT) 추가 시 enum 확장. |
| (session, account) UNIQUE upsert | 같은 학생 같은 세션 중복 row 차단 + 멱등 update. | announcement_read 와 같은 패턴. |
| recorded_by_account_id 필수 | 누가 입력했는지 audit (TEACHER vs OWNER 구분). | 분쟁 / 학부모 문의 시 추적. |
| 출결 입력 권한 TEACHER + OWNER | 강사 부재 시 학원장 대체 입력. 책임 분리는 recorded_by 로. | controller path 는 TEACHER 전용 (현재). OWNER path 별도 분리는 B6C+. |
| session 시간 end > start invariant | 도메인 메서드 (Builder + update) 가 강제. 잘못된 시간 입력 차단. | DB CHECK 제약 (MySQL 8.0.16+) 도 가능. application 일관. |
| IllegalArgumentException → 400 invalid_argument | 도메인 invariant 위반은 클라가 보낸 데이터 문제 → 400. | 도메인 한정 예외 (SessionInvalidTime) 도 가능. IllegalArgument 가 단순. |
| 자동 결석 처리 X (명시 입력만) | "강사가 입력 안 함" 의 의미가 모호 (수업 안 함 / 깜빡 / 기록 안 함). 명시만 신뢰. | scheduler 가 결석 자동 입력은 정책 결정 후 (B7+). |
| Owner controller path 분리 (`/courses/{cid}/sessions` vs `/sessions/{sid}`) | session 생성은 course 컨텍스트 필요, 수정/삭제는 session ID 만 충분. | 한 path 로 통합 시 raw session ID 가 academy 외부 노출. |
| Teacher controller 통합 (sessions + attendance) | 강사 화면이 일정 보고 곧바로 출결 입력 — 같은 도메인. | 분리하면 controller N 개 누적. |
| Student controller 통합 + role 무관 | 학생/강사/학원장이 본인 일정 GET 가능 (자기 데이터만). | 명시 @PreAuthorize STUDENT_* 도 가능. Resolver 가 학원 미가입 시 빈 응답. |
| 11개 기존 테스트 cleanup 보강 | V13 FK 누적으로 course → teacher_assignment 순서 의존 grow. 명시 정리. | BaseAcademyTest 헬퍼 추출 후보 (V14+ 시점 결정). |

# 다음 단계 (예정)

- B2A → ... → B6B 순으로 사용자 manual merge → CI 자동 실행
- B6C 시작 — 강사 권한 매트릭스 정밀화 (강사가 학생 데이터 / 출결 / 수업 정보 수정 권한)
- B7+ — 학습 콘텐츠 (생기부 / 자소서 / 예상질문 / 기출 / 모의면접 ...)

# 회수 / pitfall (누적)

- V13 FK 누적 → 11 테스트 cleanup 의 course/session/attendance 추가. FK 의존 역순 cleanup 패턴이 5+ FK 시점부터 부담 — BaseAcademyTest 헬퍼 추출 강력 후보.

# 참고

- [[../decisions/decision-session-attendance-model]]
- [[task-log-b6a-course-enrollment]]
