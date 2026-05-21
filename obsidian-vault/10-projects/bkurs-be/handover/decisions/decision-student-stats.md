---
title: "B11 결정 — Student stats 7 영역 aggregation"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T20:05:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, stats, b11]
related:
  - "[[../bkurs-be]]"
  - "[[decision-record-bridge]]"
home_hub: bkurs-be
---

# B11 결정 — Student stats

## 결론

1. **7 영역 aggregation** — Attendance/Homework/Performance/Discussion/Reading/Simulation/Record.
2. **Java 계산 (findAll → reduce)** — PoC 단순 / 후속 @Query COUNT.
3. **3 endpoint** — STUDENT 본인 / TEACHER enrolled / OWNER 본인 학원 학생.
4. **null 처리** — 데이터 없으면 카운트 0, rate 는 null (NaN 회피).
5. **카테고리 카운트** — 수동 record + bridge record 합산 (sourceType 무관).

## 왜 필요한가

- 학생 활동이 6 도메인 + 1 영구 record 에 분산 → 정량 통계 불가.
- 생기부 자료 작성 시 "토론 N회 참여, 평균 90점" 수준 정량 표현 필요.
- 학원장이 학원 학생 활동 한눈에.

## 4 대안 — 계산 위치

### A. Java service findAll + reduce — **채택 (1차)**

- 장점: 의존성 없음, debug 쉬움.
- 단점: 데이터 누적 시 메모리 / N+1 가능.
- **채택**.

### B. @Query COUNT/AVG SQL

- 장점: 성능.
- 단점: 7 영역 × 다수 metric → query 폭증.
- **기각** (1차): 후속 PR 최적화.

### C. DB view / materialized view

- **기각**: MySQL view 는 갱신 비용.

## 4 대안 — 단일 vs 분할 endpoint

### A. 단일 endpoint (7 영역 통합 응답) — **채택**

- 장점: 클라이언트 1회 호출.

### B. 영역 별 7 endpoint

- **기각**: 클라이언트 7회 호출 + N+1.

## endpoint 설계

| method | path | role |
|---|---|---|
| GET | /api/v1/students/me/stats | STUDENT |
| GET | /api/v1/teachers/me/students/{sid}/stats | TEACHER |
| GET | /api/v1/academies/me/students/{sid}/stats | OWNER |

## StudentStatsView shape

```json
{
  "studentAccountId": 123,
  "attendance": {"total": 20, "present": 18, "absent": 1, "late": 1, "excused": 0, "rate": 0.9},
  "homework": {"total": 5, "submitted": 4, "graded": 3, "avgScore": 85.0},
  "performance": {"total": 3, "submitted": 3, "graded": 2, "avgScore": 92.5},
  "discussion": {"posts": 4, "evaluated": 3, "avgScore": 88.0},
  "reading": {"notes": 5, "feedbackReceived": 4, "avgRecommended": 4.0},
  "simulation": {"responses": 2, "reviewed": 2, "avgFeasibility": 4.5},
  "record": {"total": 12, "byCategory": {"AWARD": 1, "SUBJECT": 3, "READING": 4, ...}}
}
```

## 회귀 10

1. 학생 본인 stats — 7 영역 다 0 (no data)
2. attendance 8/10 = 0.8 rate
3. homework 3 submitted / 2 graded / avg 85
4. PE 2 submitted / 1 graded / avg 92
5. discussion 3 posts / 2 evaluated / avg 88
6. reading 4 notes / 2 feedback / avg 3.5
7. simulation 1 response / 1 reviewed / fs 5
8. record by category 합산 (수동 + bridge)
9. TEACHER enrolled 학생 stats — 본인 데이터 매치
10. 다른 학원 학생 → 403

## 관련

- [[decision-record-bridge]] — record 카테고리 합산
- [[decision-unified-search]] — 검색 결과와 stats 는 별도 (검색 = 본문, stats = 카운트)
- 후속: @Query 최적화 / time-series / 학원 rollup
