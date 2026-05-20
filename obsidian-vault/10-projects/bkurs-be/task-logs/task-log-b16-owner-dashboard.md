---
title: "B16 task-log — 학원장 dashboard 학원 rollup"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T21:05:00+09:00
agent: engineering-agent/coding
tags: [task-log, dashboard, b16]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-owner-dashboard]]"
home_hub: bkurs-be
---

# B16 task-log — 학원장 dashboard

## 결과

- 이슈: #59
- PR: #60 (draft, stacked on #58 B13)
- 브랜치: `feature/issue-59-b16-owner-dashboard`
- 3 commit (S1+S2 / S3 / S4)
- 8 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| 2f24119 | DTO + Service (9 지표) | +155 | 학원 전체 집계 |
| fc3c32f | endpoint | +24 | OWNER role |
| d379a69 | AcademyDashboardApiTest 8 + CHANGELOG | +268 | 모든 회귀 PASS |

## 1 endpoint

- OWNER — GET `/api/v1/academies/me/dashboard`

## 9 지표

| 지표 | 측정 |
|---|---|
| activeStudentCount | ACTIVE membership |
| activeTeacherCount | ACTIVE teacher assignment |
| courseCount | academy 의 모든 course |
| totalSessionCount | 모든 수업 session 합 |
| totalEnrollmentCount | ACTIVE enrollment 합 |
| avgAttendanceRate | PRESENT / total (학원 전체) |
| totalGradedHomeworkCount | grade != null |
| totalGradedPECount | score != null |
| totalRecordCount | 학원 학생들의 record 합 |

## 8 회귀 (AcademyDashboardApiTest)

1. 기본 (1 학생 / 1 강사 / 1 수업) ✅
2. activeStudentCount REJECTED 제외 ✅
3. multi-course enrollment count ✅
4. session + attendance rate 0.5 ✅
5. graded homework count ✅
6. graded PE count ✅
7. 다른 학원 데이터 격리 ✅
8. STUDENT → 4xx ✅

## 선택 이유 — handover

### A. 학원 단위 통합 (수업/학생 list X)

- **선택 이유**: 1 응답 충분 — 학원장이 운영 결정에 필요한 폭.
- **기각된 대안**: 수업 list (B13 와 중복) / 학생 list (B11 stats 와 중복).
- **trade-off**: 상세는 OWNER 가 각 학생 stats / 강사 dashboard 별도 조회.

### B. AcademyManagementService.getMine (1인 1학원 검증)

- **선택 이유**: 학원 격리 자동 — academyId 기반 query.
- **trade-off**: 학원 없는 OWNER → 예외 (이미 NotFound handler 있음).

### C. activeStudentCount ACTIVE 만 (REJECTED 제외)

- **선택 이유**: 운영 결정 자료 — 거절된 신청은 무관.
- **기각된 대안**: 모든 membership 카운트 (PENDING/REJECTED 포함) — 의미 불명.

### D. avgAttendanceRate 학원 전체 (수업/학생 무관)

- **선택 이유**: 학원 단위 1 수치.
- **trade-off**: 수업 별 / 학생 별 출석률은 B13/B11.

### E. N+1 PoC (후속 @Query 최적화)

- **선택 이유**: B11/B13 패턴 일관, debug 쉬움.
- **future**: 학원 누적 시 SQL aggregate query 로 전환.

## 관련

- [[../decisions/decision-owner-dashboard]]
- [[task-log-b13-teacher-dashboard]] — 강사 1인 수업 시야
- [[task-log-b11-student-stats]] — 학생 1:1 시야
- 후속: time-series / 학원 비교 (MASTER) / 알림 / B17
