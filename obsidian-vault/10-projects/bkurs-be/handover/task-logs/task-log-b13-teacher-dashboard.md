---
title: "B13 task-log — 강사 dashboard"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T20:55:00+09:00
agent: engineering-agent/coding
tags: [task-log, dashboard, b13]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-teacher-dashboard]]"
home_hub: bkurs-be
---

# B13 task-log — 강사 dashboard

## 결과

- 이슈: #57
- PR: #58 (draft, stacked on #56 B11)
- 브랜치: `feature/issue-57-b13-teacher-dashboard`
- 3 commit (S1+S2 / S3 / S4)
- 8 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| f26c38a | DTO + Service (수업별 5 지표) | +130 | 수업 loop + Java reduce |
| 9cf1bfd | endpoint | +24 | TEACHER role |
| 5f3d58c | TeacherDashboardApiTest 8 + CHANGELOG | +268 | 모든 회귀 PASS |

## 1 endpoint

- TEACHER — GET `/api/v1/teachers/me/dashboard`

## 5 지표 (수업별)

| 지표 | 측정 |
|---|---|
| enrolledCount | ACTIVE enrollment 만 |
| pendingHomeworkCount | grade=null submission |
| pendingPECount | score=null submission |
| sessionCount | 총 session 수 |
| avgAttendanceRate | PRESENT / total (학생 무관 합산) |

## 8 회귀 (TeacherDashboardApiTest)

1. 강사 dashboard — 본인 배정 수업 1개 ✅
2. 빈 dashboard (배정 없음) ✅
3. enrolledCount ACTIVE 만 (DROPPED 제외) ✅
4. pendingHomework grade=null 만 ✅
5. pendingPE score=null 만 ✅
6. avgAttendanceRate = 0.5 (1 PRESENT / 2 total) ✅
7. 다른 강사 수업 안 보임 (격리) ✅
8. STUDENT → 4xx ✅

## 선택 이유 — handover

### A. 수업 별 + 5 지표 (학생 별 차원 X)

- **선택 이유**: 1 round-trip, 클라이언트 UI 단순.
- **기각된 대안**: 수업 × 학생 (2 차원) — 응답 폭증, N+1.
- **trade-off**: 학생 별 상세는 B11 stats endpoint (클릭 후 별도 호출).

### B. pending 정의 = grade/score null

- **선택 이유**: 명확.
- **기각된 대안**: N일 경과 (시간 기준 임의).

### C. avgAttendanceRate 학생 무관 합산

- **선택 이유**: 수업 단위 출석률 — 어느 학생인지 dashboard 에서 무의미.
- **기각된 대안**: 학생별 출석률 list — 응답 복잡.
- **trade-off**: 학생별은 B11 stats.

### D. STUDENT 4xx (TEACHER role 만)

- **선택 이유**: dashboard = 강사 전용.
- **trade-off**: 학원장은 별도 OWNER dashboard (후속).

### E. C1+C2 single commit (4 commit 패턴 deviation)

- **선택 이유**: DTO 2 record + service 1 — 한 commit 으로 자연 (분리 인위적).
- **결과**: 3 commit (DTO+service / endpoint / test) — 최소 3 commit 충족.

## 관련

- [[../decisions/decision-teacher-dashboard]]
- [[task-log-b11-student-stats]] — 학생 1:1 stats
- 후속: 알림 / 시간대 추이 / OWNER 학원 dashboard / B12 period
