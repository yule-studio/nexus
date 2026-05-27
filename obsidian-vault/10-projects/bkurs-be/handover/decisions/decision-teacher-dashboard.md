---
title: "B13 결정 — 강사 dashboard 수업별 진도"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T20:50:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, dashboard, b13]
related:
  - "[[../bkurs-be]]"
  - "[[decision-student-stats]]"
home_hub: bkurs-be
---

# B13 결정 — 강사 dashboard

## 결론

1. **수업 별 dashboard (`CourseDashboard` sub-record)** — 학생 별 stats 는 B11 위임.
2. **5 지표** — enrolledCount / pendingHomeworkCount / pendingPECount / sessionCount / avgAttendanceRate.
3. **1 endpoint** (`GET /api/v1/teachers/me/dashboard`) — 본인 배정 모든 수업.
4. **Java 계산 (findAll + reduce)** — B11 패턴 일관, 후속 @Query 최적화.

## 왜 필요한가

- 강사가 본인 우선순위 (채점 밀린 수업) 빠르게 식별.
- B11 학생 stats 는 1:1 (학생 클릭 후 상세) — dashboard 는 1:N (수업 한눈에).

## 4 대안 — 깊이

### A. 수업 별 + 5 지표 — **채택**

- 장점: 1 round-trip, 클라이언트 UI 단순.

### B. 수업 별 + 학생 별 (2 차원)

- **기각**: 응답 폭증, N+1 위험. 학생 별 상세는 B11.

### C. 단순 카운트만 (총 수업 / 총 학생)

- **기각**: 우선순위 식별 어려움 (어느 수업 채점 밀렸는지 불명).

## 4 대안 — pending 정의

### A. grade=null / score=null — **채택**

- 장점: 명확.
- **채택**.

### B. submittedAt 이후 N일 경과

- **기각**: 시간 기준 임의.

## endpoint 설계

| method | path | role |
|---|---|---|
| GET | /api/v1/teachers/me/dashboard | TEACHER |

## TeacherDashboardView shape

```json
{
  "teacherAccountId": 123,
  "courses": [
    {
      "courseId": 1,
      "courseName": "수학",
      "enrolledCount": 12,
      "pendingHomeworkCount": 3,
      "pendingPECount": 1,
      "sessionCount": 20,
      "avgAttendanceRate": 0.92
    }, ...
  ]
}
```

## 회귀 8

1. 강사 dashboard — 본인 배정 수업 모두 노출
2. 빈 dashboard (배정 수업 없음)
3. enrolledCount 정확 (ACTIVE만 카운트)
4. pendingHomeworkCount = 미채점 수
5. pendingPECount = 미채점 수
6. avgAttendanceRate = PRESENT / total
7. 다른 강사 수업 안 보임 (격리)
8. STUDENT/OWNER → 4xx

## 관련

- [[decision-student-stats]] — 학생 별 상세 stats (1:1)
- 후속: 강사 알림 (pending > N 알림) / 시간대 별 출결 추이
