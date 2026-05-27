---
title: "B7A — Homework + Submission + 채점 작업 로그 (학습 콘텐츠 시작)"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b7-content
created_at: 2026-05-20T12:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-homework-submission-model]]"
  - "[[task-log-b6c-teacher-permission]]"
tags: [task-log, homework, submission, b7a]
---

# 작업

GitHub issue [#31](https://github.com/seum-running/bkurs-be/issues/31) — Homework + Submission baseline.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 모델 | homework + homework_submission 별도 (정규화) | [[../decisions/decision-homework-submission-model]] |
| upsert | (homework, account) UNIQUE → 학생 재제출 update | content 갱신, 채점 유지 |
| 채점 | TEACHER 본인 배정 수업 (OWNER 도 패턴 동일) | grade 0-100 invariant, graded_by audit |
| due_at | nullable, enforce X (학원 정책) | 향후 옵션 |
| 권한 패턴 | B6 의 강사 본인 배정 검증 재사용 | CourseService.listByTeacher |

# 진행

## 2026-05-20

### 1. branch / issue
- B6C 머지 대기 중 stacked PR — base=feature/issue-29-b6c-teacher-permission
- issue #31 + decision 노트

### 2. 코드 (commit 단위)
- C1: Homework + Submission entity + V14 + invariant (title/content/grade 0-100)
- C2: 3 예외 + HomeworkService (강사 CRUD + ensureTeacherOfCourse) + SubmissionService (학생 upsert + 강사 채점 + 본인 수강 검증)
- C3: 6 DTO + 2 controller (Teacher 6 endpoint / Student 3 endpoint) + ExceptionHandler
- C4: HomeworkApiTest 10 + 14 기존 테스트 cleanup 보강 + CHANGELOG B7A

### 3. PR
- PR [#32](https://github.com/seum-running/bkurs-be/pull/32) (feature/issue-31 → feature/issue-29-b6c-teacher-permission, stacked)
- 회귀 **142/142 GREEN** — **B7 시리즈 시작**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| homework + submission 정규화 | array JSON 안티패턴 회피. 양방향 query (학생별 / 숙제별). | 추후 채점 통계 / 학원 평균 / 정렬 모두 SQL. |
| (homework, account) UNIQUE upsert | 학생 재제출 = update. announcement_read / attendance 와 같은 패턴. | 재제출 이력 보존 필요 시 별도 submission_revision 테이블. |
| resubmit 시 채점 정보 유지 | 학생 재제출했다고 강사 채점 reset 은 어색 — 강사가 명시 재채점 필요. | 강사가 "재채점 필요" 마킹 옵션 (B7+) 검토. |
| grade 0-100 invariant | 학원 평가 표준. entity가 강제. | 다른 평가 체계 (P/F, A-F) 도입 시 enum + integer scale 검토. |
| due_at nullable + enforce X | 학원 운영 다양 — 일부 학원은 due 강제, 일부는 자율. 정책 layer 분리. | enforce 옵션은 학원 settings 에 추가 (B4+ 확장). |
| TEACHER 권한 = 본인 배정 수업 | ensureTeacherOfCourse — CourseService.listByTeacher 위임. B6 패턴 재사용. | OWNER 대체 출제 / 채점은 controller path 추가 (현재 service 는 미허용). |
| 학생 본인 수강 검증 in service | controller 가 학원 검증 어려움 (course → academy chain). service 가 enrollment 직접 검증. | AcademyMembershipResolver 도 추가로 통합 가능하지만 enrollment 직접이 더 정확. |
| Teacher controller path /teachers/me + Student /students/me | role 분리 명확. | OWNER path 별도 추가 (향후). |
| 14 테스트 cleanup 보강 — homework/submission deleteAll | V14 FK (assigned_by + account) NOT CASCADE → 명시 정리 필수. | BaseAcademyTest 헬퍼 추출 강한 후보 — 14 테스트 동일 패턴. |
| SubmissionService.myActiveEnrolledCourseIds 헬퍼 | StudentController 가 enrollment 직접 조회는 service layer 책임 흐림. helper 로 응집. | AcademyMembershipResolver 와 비슷한 cross-cutting — B7+ 통합 검토. |

# 다음 단계 (예정)

- B7B 자소서 (PersonalStatement) — 학생 작성 / 강사 피드백
- B7C 예상질문 / 기출 — 강사/학원장 출제 + 학생 풀이
- B7D 모의면접 — 학생 영상/텍스트 + 강사 평가
- B7E 생기부 (StudentRecord) — 학원/강사 평가 누적 + PDF export
- B7F 토론 / 수행평가 / 독서 / 시뮬레이션

# 회수 / pitfall (누적)

- **14 테스트 cleanup 반복** — V14 누적으로 동일 cleanup 패턴 14 곳 복제. BaseAcademyTest 헬퍼 추출 강력 후보 (vault TODO). B7B+ 도입 전 우선순위.

# 참고

- [[../decisions/decision-homework-submission-model]]
- [[task-log-b6c-teacher-permission]]
