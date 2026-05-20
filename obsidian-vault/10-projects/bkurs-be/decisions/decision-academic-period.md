---
title: "B12 결정 — 학기 자동 결정 (한국 학년도)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T21:15:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, academic-period, b12]
related:
  - "[[../bkurs-be]]"
  - "[[decision-record-bridge]]"
home_hub: bkurs-be
---

# B12 결정 — 학기 자동 결정

## 결론

1. **한국 학년도 규칙**: 학년도 = 3월 of N ~ 2월 of N+1.
2. **`AcademicPeriod.from(LocalDate)` 정적 유틸** — pure function, side effect 없음.
3. **`StudentRecordBridgeService` 통합** — bridge 시 today 기준 period 자동 설정.
4. **8월 = 2학기** (보통 8월말 개학 관례) — 결정 노트 명시.
5. **format: `"{학년도}-{학기}학기"`** (예: `"2026-1학기"`).

## 한국 학년도 매핑

| 월 (today) | 학기 | 학년도 | format |
|---|---|---|---|
| 3-7 | 1 | year | "year-1학기" |
| 8-12 | 2 | year | "year-2학기" |
| 1-2 | 2 | year-1 | "(year-1)-2학기" |

## 4 대안 — 8월 처리

### A. 2학기로 분류 — **채택**

- 8월말 개학 관례.
- **채택**.

### B. 1학기로 분류

- 여름방학 (1학기 기말 후 직후) 으로 보면 1학기 연장.
- **기각**: 9월 개학 후 분류 일관성.

### C. 미정 (null)

- **기각**: bridge 가 학년도/학기 결정 못함.

## 4 대안 — 2월 처리 (전년도 2학기)

### A. 전년도 2학기 — **채택**

- 학년도 = 3월 of N ~ 2월 of N+1 일관.
- **채택**.

### B. 당해 1학기 (3월 전 = 학기 시작 전)

- **기각**: 학년도 경계 모호.

## 4 대안 — 사용자 override

### A. 수정 endpoint (StudentRecord PATCH) 이용 — **채택**

- 장점: 기존 endpoint 재사용. UI/UX 일관.

### B. 강사가 period 직접 지정 endpoint

- **기각**: bridge 자동화 가치 약화.

## 적용 위치

- `StudentRecordBridgeService.upsert()` — 새 record 생성 시 `AcademicPeriod.from(LocalDate.now(KST)).format()`.
- 기존 record (period=null) 는 유지 — DB migration 없음.

## 회귀

### unit (AcademicPeriodTest)

1. 3월 → 1학기 당해
2. 7월 → 1학기 당해
3. 8월 → 2학기 당해
4. 12월 → 2학기 당해
5. 1월 → 2학기 전년도
6. 2월 → 2학기 전년도
7. format 정확성
8. KST 기준 (UTC 시차 9시간 자정 경계)

### integration (StudentRecordBridgeApiTest)

- 기존 회귀에 `assertThat(record.period).matches("\\d{4}-[12]학기")` 추가.

## 관련

- [[decision-record-bridge]] — bridge 가 record 생성 시점 trigger
- 후속: 학년도 cutoff 사용자 설정 (3월 외 시작) / period 별 통계
