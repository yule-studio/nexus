---
title: "B11 task-log — 학생 통계 7 영역"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T20:40:00+09:00
agent: engineering-agent/coding
tags: [task-log, stats, b11]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-student-stats]]"
home_hub: bkurs-be
---

# B11 task-log — 학생 통계

## 결과

- 이슈: #55
- PR: #56 (draft, stacked on #54 B10)
- 브랜치: `feature/issue-55-b11-student-stats`
- 4 commit (S1~S4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| 9da3937 | DTO 7 sub-record + StudentStatsView + 예외 | +90 | empty() factory |
| d37e5de | StatsService 7 영역 aggregation | +209 | findAll + Java reduce |
| c666d9f | 3 controller + ExceptionHandler | +94 | TEACHER/STUDENT/OWNER |
| 81d1845 | StudentStatsApiTest 10 + CHANGELOG | +380 | URL 오타 수정 포함 |

## 3 endpoint

- STUDENT — GET `/api/v1/students/me/stats`
- TEACHER — GET `/api/v1/teachers/me/students/{sid}/stats`
- OWNER — GET `/api/v1/academies/me/students/{sid}/stats`

## 7 영역 지표

| 영역 | DTO | 지표 |
|---|---|---|
| Attendance | AttendanceStats | total/present/absent/late/excused + rate |
| Homework | HomeworkStats | total/submitted/graded + avgScore |
| Performance | PerformanceStats | total/submitted/graded + avgScore |
| Discussion | DiscussionStats | posts/evaluated + avgScore |
| Reading | ReadingStats | notes/feedbackReceived + avgRecommended |
| Simulation | SimulationStats | responses/reviewed + avgFeasibility |
| Record | RecordStats | total + EnumMap byCategory |

## 10 회귀 (StudentStatsApiTest)

1. empty stats — 7 영역 모두 0 / null ✅
2. attendance 4/5 = 0.8 ✅
3. homework graded avg 80 ✅
4. PE graded avg 90 ✅
5. discussion 3 posts / 2 evaluated avg 85 ✅
6. reading 2 notes / 1 fb rec 4 ✅
7. simulation 1 reviewed fs 5 ✅
8. record bridge + 수동 합산 (SUBJECT/READING/AWARD) ✅
9. TEACHER enrolled stats ✅
10. 다른 학원 학생 403 ✅

## 선택 이유 — handover

### A. Java 계산 (@Query COUNT 보류)

- **선택 이유**: 의존성 없음 + debug 쉬움.
- **기각된 대안**: @Query SQL COUNT/AVG — 7 영역 × 다수 metric → query 폭증.
- **trade-off**: N+1, 데이터 누적 시 메모리. 후속 최적화.

### B. 단일 endpoint (7 영역 통합 응답)

- **선택 이유**: 클라이언트 1회 호출 + 응답 일관성.
- **기각된 대안**: 영역 별 7 endpoint (클라이언트 7회 호출).

### C. null rate (NaN 회피)

- **선택 이유**: total=0 일 때 rate = present/0 = NaN 회피.
- **기각된 대안**: rate=0 (의미 불명, "0% 출석" 으로 오해 가능).
- **convention**: 데이터 없으면 null → 클라이언트가 "—" 표시.

### D. findBy...Containing("") 트릭으로 학생별 전체 조회

- **선택 이유**: 별도 finder (findByStudentAccountIdOrderBy...) 추가 회피.
- **기각된 대안**: 5 repo 각각 findByStudentAccountId 메서드 추가.
- **trade-off**: 의도가 코드에서 불명확 — comment 로 보완.

### E. record 카테고리 합산 (bridge + 수동 무관)

- **선택 이유**: 생기부 자료는 정량 — source 구분 불필요.
- **기각된 대안**: bridge / manual 분리 표시 (UI 복잡).
- **future**: 필요 시 별도 endpoint 로 source 분리.

## 관련

- [[../decisions/decision-student-stats]]
- [[task-log-b9-record-bridge]] — record 카테고리 합산
- [[task-log-b10-search]] — 검색 (본문) vs stats (카운트) 별도
- 후속: @Query 최적화 / time-series / 학원 rollup / B12 period
