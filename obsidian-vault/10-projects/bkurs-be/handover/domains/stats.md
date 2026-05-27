---
title: "도메인 — stats (학생 7 영역 통계)"
kind: handover
status: stable
created_at: 2026-05-21T23:57:00+09:00
tags: [bkurs-be, handover, domain, stats]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-student-stats]]"
home_hub: "[[../../bkurs-be]]"
---

# stats 도메인

> 한 학생의 7 영역 활동 통계 통합. 엔티티 / migration 없음 — aggregator. **현재 findAll + Java
> reduce** (후속 @Query COUNT 최적화 후보).

## 1. 책임

- 7 sub-record 통계: Attendance / Homework / Performance / Discussion / Reading / Simulation / Record
- 통합 `StudentStatsView` 조립 (각 영역 카운트 / 평균 / 비율)
- role 별 endpoint 3개

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `StudentStatsView` (record) | 7 sub-record 통합 |
| `AttendanceStats` | total / present / late / excused / absent / rate |
| `HomeworkStats` | total / submitted / graded / avgGrade |
| `PerformanceStats` | total / submitted / avgGrade / byCategory map |
| `DiscussionStats` | postsCount / avgEvaluation |
| `ReadingStats` | notesCount / avgRecommendedScore |
| `SimulationStats` | responsesCount / avgFeasibility |
| `RecordStats` | totalEntries / byCategory map (MANUAL + bridge 합산) |

## 3. 패키지 구성

```
stats/
├── controller/
│   ├── StudentStatsController       # /api/v1/students/me/stats
│   ├── TeacherStatsController       # /api/v1/teachers/me/students/{id}/stats
│   └── OwnerStatsController         # /api/v1/academies/me/students/{id}/stats
├── service/                         # StatsService — 7 영역 aggregator
├── dto/{request,response}/
├── enums/                           # StatsErrorCode (STS-)
├── exception/
└── handler/
```

## 4. API endpoint

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| GET | `/api/v1/students/me/stats` | STUDENT_* | 본인 통계 |
| GET | `/api/v1/teachers/me/students/{studentId}/stats` | TEACHER | 학생 통계 |
| GET | `/api/v1/academies/me/students/{studentId}/stats` | OWNER | 학원 학생 통계 |

## 5. 핵심 코드 흐름

```
StatsService.collectForStudent(studentId)
  ▼
AttendanceStats a = computeAttendance(studentId);
HomeworkStats   h = computeHomework(studentId);
PerformanceStats p = computePerformance(studentId);
DiscussionStats  d = computeDiscussion(studentId);
ReadingStats     r = computeReading(studentId);
SimulationStats  s = computeSimulation(studentId);
RecordStats      rec = computeRecord(studentId);
return new StudentStatsView(a, h, p, d, r, s, rec);
```

각 compute 메서드는 해당 도메인 repository 의 findAll + stream reduce.

## 6. 한계 / 후속

- 학생 1명 stats 호출 시 7 query — 학생 많아지면 부담.
- 후속: 각 repository 에 `@Query` COUNT / AVG / GROUP BY 추가하여 stats query 5~7 로 통합.

## 7. 의존

- **의존**: course (attendance), homework, performance, discussion, reading, simulation, record (Repository 7개)

## 서비스 로직 (`stats/service/`)

1 service. 7 영역 aggregator + role 별 권한 분기.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `StatsService` | `@Transactional(readOnly)` | 7 sub-record findAll + Java reduce |

### 핵심 메서드

```java
public StudentStatsView statsForStudent(Long studentAccountId)
                                                          // 학생 본인 통계
public StudentStatsView statsForTeacher(Long teacherAccountId, Long studentAccountId)
                                                          // 강사 본인 수업 학생 검증 → 통계
public StudentStatsView statsForOwner(Long ownerAccountId, Long studentAccountId)
                                                          // 학원장 본인 학원 학생 검증 → 통계
```

내부 (3 메서드 공통, 권한 검증만 다름):

```java
AttendanceStats   a   = computeAttendance(studentId);    // total / present / late / excused / absent / rate
HomeworkStats     h   = computeHomework(studentId);      // total / submitted / graded / avgGrade
PerformanceStats  p   = computePerformance(studentId);   // total / submitted / avgGrade / byCategory
DiscussionStats   d   = computeDiscussion(studentId);    // postsCount / avgEvaluation
ReadingStats      r   = computeReading(studentId);       // notesCount / avgRecommendedScore
SimulationStats   s   = computeSimulation(studentId);    // responsesCount / avgFeasibility
RecordStats       rec = computeRecord(studentId);        // totalEntries / byCategory (MANUAL+bridge 합산)
return new StudentStatsView(a, h, p, d, r, s, rec);
```

각 compute 메서드는 해당 도메인 repository findAll + stream reduce — 학생 1명 stats = 7 query. 후속 @Query COUNT/AVG 최적화 후보.

## cross-link

- [[dashboard]] — 학원 단위 rollup (별도)
- [[../decisions/decision-student-stats]]
