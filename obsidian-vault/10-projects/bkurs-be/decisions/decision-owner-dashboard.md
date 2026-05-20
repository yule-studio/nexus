---
title: "B16 결정 — 학원장 dashboard 학원 단위 rollup"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T21:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, dashboard, b16]
related:
  - "[[../bkurs-be]]"
  - "[[decision-teacher-dashboard]]"
home_hub: bkurs-be
---

# B16 결정 — 학원장 dashboard

## 결론

1. **학원 전체 rollup (9 지표)** — 학생/강사/수업/세션/enrollment/출결/채점/record.
2. **1 endpoint** (`GET /api/v1/academies/me/dashboard`).
3. **Java 계산 (findBy + reduce)** — B11/B13 패턴 일관.
4. **학원 미보유 OWNER** — 빈 dashboard (전부 0 / null).

## 왜 필요한가

- 학원 운영 결정 (강사 추가 / 학생 모집 / 채점 catch-up) 정량 자료.
- B13 강사 dashboard 는 강사 1인 시야, B16 은 학원장 전체.

## 4 대안 — 깊이

### A. 학원 단위 통합 — **채택**

- 장점: 1 응답, 운영 결정에 충분.

### B. 학원 단위 + 수업 별 list

- **기각**: B13 (강사 dashboard) 와 중복. OWNER 도 본인 학원 강사들의 dashboard 별도 확인.

### C. 학원 단위 + 학생 별 list

- **기각**: B11 stats endpoint 가 학생별 (per-OWNER).

## 9 지표

| 지표 | source |
|---|---|
| activeStudentCount | StudentAcademyMembership ACTIVE |
| activeTeacherCount | TeacherAcademyAssignment ACTIVE |
| courseCount | Course.academyId |
| totalSessionCount | CourseSession (학원 수업 합) |
| totalEnrollmentCount | CourseEnrollment ACTIVE (학생-수업 페어 수) |
| avgAttendanceRate | PRESENT / total (학원 전체) |
| totalGradedHomeworkCount | HomeworkSubmission grade != null |
| totalGradedPECount | PerformanceSubmission score != null |
| totalRecordCount | StudentRecordEntry (학원 학생들의) |

## endpoint 설계

| method | path | role |
|---|---|---|
| GET | /api/v1/academies/me/dashboard | OWNER |

## 회귀 8

1. 학원장 dashboard — 모든 지표 기본 값
2. activeStudentCount 정확 (REJECTED 제외)
3. activeTeacherCount 정확
4. courseCount + totalSessionCount
5. totalEnrollmentCount
6. avgAttendanceRate
7. totalGradedHomeworkCount + totalGradedPECount
8. 다른 학원 영향 없음 + STUDENT 거부

## 관련

- [[decision-teacher-dashboard]] — 강사 1인 시야 (1:N 수업)
- [[decision-student-stats]] — 학생 1:1 (per-student deep)
- 후속: time-series / 학원 비교 (MASTER 권한) / 알림
