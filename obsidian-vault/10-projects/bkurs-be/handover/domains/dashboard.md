---
title: "도메인 — dashboard (강사 수업별 / 학원장 학원 rollup)"
kind: handover
status: stable
created_at: 2026-05-21T23:58:00+09:00
tags: [bkurs-be, handover, domain, dashboard]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-teacher-dashboard]]"
  - "[[../decisions/decision-owner-dashboard]]"
home_hub: "[[../../bkurs-be]]"
---

# dashboard 도메인

> stats 와 구분: 본 도메인은 **수업 / 학원 단위 rollup**, stats 는 **학생 1명 단위**. 강사
> 대시보드는 본인 배정 수업 loop, 학원장 대시보드는 학원 전체 집계.

## 1. 책임

- **강사**: 본인 배정 수업 각각의 5 지표 (`CourseDashboard`) 모음 → `TeacherDashboardView`
- **학원장**: 본인 학원의 9 지표 (학원 전체 rollup) → `AcademyDashboardView`

## 2. 핵심 타입

### TeacherDashboardView

수업 1개당 `CourseDashboard` 5 지표:

| 지표 | 의미 |
| --- | --- |
| `enrolledCount` | ACTIVE enrollment 수 |
| `pendingHomeworkCount` | 미제출 + 미채점 숙제 수 |
| `pendingPECount` | 미채점 수행평가 수 |
| `sessionCount` | 수업 회차 수 |
| `avgAttendanceRate` | 평균 출석률 (0~1) |

### AcademyDashboardView (9 지표)

| 지표 |
| --- |
| `activeStudentCount` |
| `activeTeacherCount` |
| `courseCount` |
| `sessionCount` |
| `enrollmentCount` |
| `attendanceCount` |
| `gradedHomeworkCount` |
| `gradedPECount` |
| `recordCount` |

empty factory 메서드 — 학원 없거나 데이터 없으면 모두 0 으로 반환.

## 3. 패키지 구성

```
dashboard/
├── controller/
│   ├── TeacherDashboardController   # /api/v1/teachers/me/dashboard
│   └── OwnerDashboardController     # /api/v1/academies/me/dashboard
├── service/                         # TeacherDashboardService / AcademyDashboardService
├── dto/{request,response}/          # CourseDashboard / TeacherDashboardView / AcademyDashboardView
└── handler/
```

> entity / Repository 없음 — 다른 도메인의 Repository 만 의존.

## 4. API endpoint

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| GET | `/api/v1/teachers/me/dashboard` | TEACHER | 본인 배정 수업 5 지표 mass |
| GET | `/api/v1/academies/me/dashboard` | OWNER | 학원 9 지표 rollup |

## 5. 핵심 코드 흐름

### Teacher

```
TeacherDashboardService.collectForTeacher(teacherId)
  ▼
List<Course> myCourses = courseRepo.findByTeacherAssignment_TeacherAccountIdAndStatus(...)
return myCourses.stream()
    .map(course -> new CourseDashboard(
        course.id, course.title,
        enrollmentRepo.countByCourseIdAndStatus(course.id, ACTIVE),
        homeworkRepo.countPendingByCourse(course.id),
        performanceRepo.countPendingByCourse(course.id),
        sessionRepo.countByCourseId(course.id),
        avgAttendance(course.id)))
    .toList();
```

### Owner

```
AcademyDashboardService.collectForOwner(ownerId)
  ▼
academy = academyManagementService.getMine(ownerId)  ─ 1인 1학원 검증
return new AcademyDashboardView(
    activeStudentCount = membership.countByAcademyIdAndStatus(...),
    activeTeacherCount = assignment.countByAcademyIdAndStatus(...),
    courseCount        = course.countByAcademyId(...),
    ...
);
```

## 6. 의존

- **의존**: course / homework / performance / attendance / record / academy / discussion / reading 등 모든 통계 source 도메인
- **의존받음**: (직접 endpoint 만)

## 서비스 로직 (`dashboard/service/`)

2 service — Teacher (수업별) / Owner (학원 rollup). 엔티티 / Repository 없음.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `TeacherDashboardService` | `@Transactional(readOnly)` | 본인 배정 수업 loop + 수업별 5 지표 |
| `AcademyDashboardService` | `@Transactional(readOnly)` | 본인 학원 9 지표 rollup |

### 핵심 메서드

```java
// TeacherDashboardService
public TeacherDashboardView dashboard(Long teacherAccountId) {
    List<Course> myCourses = courseRepo.findByTeacherAssignment_TeacherAccountIdAndStatus(teacherAccountId, ACTIVE);
    List<CourseDashboard> rows = myCourses.stream()
        .map(c -> new CourseDashboard(
            c.getId(), c.getName(),
            enrollmentRepo.countByCourseIdAndStatus(c.getId(), ACTIVE),
            homeworkRepo.countPendingByCourse(c.getId()),
            performanceRepo.countPendingByCourse(c.getId()),
            sessionRepo.countByCourseId(c.getId()),
            attendanceRepo.avgAttendanceRate(c.getId())))
        .toList();
    return new TeacherDashboardView(rows);
}

// AcademyDashboardService
public AcademyDashboardView dashboard(Long ownerAccountId) {
    Academy academy = academyManagementService.getMine(ownerAccountId);  // 1인 1학원 검증
    return new AcademyDashboardView(
        membershipRepo.countByAcademyIdAndStatus(academy.getId(), ACTIVE),
        assignmentRepo.countByAcademyIdAndStatus(academy.getId(), ACTIVE),
        courseRepo.countByAcademyId(academy.getId()),
        sessionRepo.countByAcademyId(academy.getId()),
        enrollmentRepo.countByAcademyId(academy.getId()),
        attendanceRepo.countByAcademyId(academy.getId()),
        homeworkRepo.countGradedByAcademyId(academy.getId()),
        performanceRepo.countGradedByAcademyId(academy.getId()),
        recordRepo.countByAcademyId(academy.getId()));
}
```

빈 학원 / 데이터 없는 경우는 `AcademyDashboardView.empty()` factory.

## cross-link

- [[stats]] — 학생 단위 vs 본 도메인 (수업/학원 단위)
- [[../decisions/decision-teacher-dashboard]]
- [[../decisions/decision-owner-dashboard]]
