---
title: "B12 task-log — 학기 자동 결정"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T21:40:00+09:00
agent: engineering-agent/coding
tags: [task-log, academic-period, b12]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-academic-period]]"
home_hub: bkurs-be
---

# B12 task-log — 학기 자동 결정

## 결과

- 이슈: #61
- PR: #62 (draft, stacked on #60 B16)
- 브랜치: `feature/issue-61-b12-academic-period`
- 4 commit (S1 / S2 / S3 / CHANGELOG fixup)
- 8 unit + 10 integration PASS

## commit 별 산출물

| commit | 내용 | LOC |
|---|---|---|
| 93061c4 | AcademicPeriod 유틸 + 8 unit | +119 |
| 69add76 | bridge service period 자동 | +11 |
| 9059c7c | integration test period 검증 | +2 |
| 598a03c | CHANGELOG B12 (fixup) | +11 |

## 한국 학년도 매핑

| 월 | 학기 | 학년도 | 예 |
|---|---|---|---|
| 3-7 | 1 | year | "2026-1학기" |
| 8-12 | 2 | year | "2026-2학기" |
| 1-2 | 2 | year-1 | "2025-2학기" (2026년 1-2월) |

## 선택 이유 — handover

### A. 8월 = 2학기

- 한국 보통 8월 말 개학 관례.
- 1학기 (3-7월) 종료 후 방학 (7월 말~8월 중순), 8월 말 2학기 개시.

### B. 1-2월 → 전년도 2학기

- 학년도 = 3월 of N ~ 2월 of N+1 일관.
- 1월 record 가 "당해 1학기" 로 가면 학년도 경계 모호.

### C. KST `Asia/Seoul`

- UTC 시차 9시간 자정 경계 회피.
- LocalDate.now(KST) 가 한국 사용자 기대치 일관.

### D. 기존 record (period=null) 보존

- DB migration 없음 — 기존 자료 그대로.
- 신규 bridge record 부터 period 자동.

### E. 사용자 override = StudentRecord PATCH 재사용

- B7E 의 PATCH endpoint 가 period 수정 지원.
- 별도 endpoint 추가 회피.

## 관련

- [[../decisions/decision-academic-period]]
- [[task-log-b9-record-bridge]] — bridge 가 trigger
- 후속: 학년도 cutoff 사용자 설정 / period 별 통계
