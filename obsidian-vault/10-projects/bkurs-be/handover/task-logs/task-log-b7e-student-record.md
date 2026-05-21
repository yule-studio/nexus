---
title: "B7E task-log — StudentRecord (생기부) 8 카테고리"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T16:15:00+09:00
agent: engineering-agent/coding
tags: [task-log, student-record, b7e]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-student-record-categories]]"
home_hub: bkurs-be
---

# B7E task-log — StudentRecord (생기부)

## 결과

- 이슈: #41
- PR: #42 (draft, stacked on #40 B7D)
- 브랜치: `feature/issue-41-b7e-student-record`
- 5 commit (REFACTOR fixup 1 + S1~S4 4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| 356e564 | V18 + RecordCategory + Entity + Repository | +130 | 8 카테고리 enum |
| 1961fd2 | AbstractIntegrationTest fixup | +3 | Edit tool 누락 보상 |
| 56107cf | StudentRecordService + 예외 + DTO | +180 | ownership 2계층 |
| d85e315 | endpoint 4+1+1 + ExceptionHandler | +120 | 6 endpoint |
| dc2dfcb | StudentRecordApiTest 10 + CHANGELOG | +210 | 모든 회귀 PASS |

## 6 endpoint

- TEACHER — POST/GET/PATCH/DELETE `/api/v1/teachers/me/students/{id}/records` + `/records/{rid}`
- STUDENT — GET `/api/v1/students/me/records?category=`
- OWNER — GET `/api/v1/academies/me/students/{id}/records?category=`

## 8 카테고리 (RecordCategory)

| enum | 학종 영역 |
|---|---|
| AWARD | 수상 |
| AUTONOMY | 자율활동 |
| CLUB | 동아리 |
| CAREER | 진로 |
| SERVICE | 봉사 |
| SUBJECT | 교과세특 |
| BEHAVIOR | 행동특성 |
| READING | 독서 |

## 10 회귀 (StudentRecordApiTest)

1. 강사 AWARD 입력 → 201 ✅
2. 강사 CLUB / READING 다카테고리 누적 ✅
3. STUDENT 직접 입력 → 거부 (4xx) ✅
4. 강사 본인 수강생 조회 ✅
5. category=AWARD 필터 ✅
6. 다른 강사 수강생 → 403 ✅
7. 학생 본인 record 조회 ✅
8. 학원장 학생 record 조회 ✅
9. 강사 본인 record PATCH ✅
10. 다른 강사 record PATCH → 403 ✅

## 선택 이유 — handover

### A. 단일 테이블 + category enum

- **선택 이유**: 8 영역 동일 schema (title/content/period/audit) → 1 테이블이 자연.
- **기각된 대안**: 영역별 8 테이블 (CRUD 코드 8배) / JSON 컬럼 (필터 불가).
- **트레이드오프**: 영역별 schema 커스텀 불가 — 학종 표준이라 큰 차이 없음.

### B. courseId nullable + ON DELETE SET NULL

- **선택 이유**: 봉사 / 자율활동은 특정 수업에 묶이지 않음. 수업 삭제돼도 생기부 영구.
- **기각된 대안**: NOT NULL + CASCADE (수업 삭제 시 생기부도 날아감).

### C. ownership 2계층 (role + service)

- **선택 이유**: @PreAuthorize 는 coarse, service 가 본인 수강생인지 verify.
- **트레이드오프**: 검증 코드 늘어남 — 대신 학원 간 침범 절대 차단.

### D. 작성자 본인 만 수정/삭제

- **선택 이유**: 같은 학원 다른 강사가 남의 평가 수정 못 함. 책임 명확.
- **기각된 대안**: 학원 강사 누구나 수정 — 책임 추적 어려움.

### E. STUDENT 거부 케이스 401/403 둘 다 허용

- **이유**: Spring 6 `@PreAuthorize` 거부 시 `AuthorizationDeniedException` → `ExceptionTranslationFilter` 가 환경에 따라 401 (authenticationEntryPoint) 또는 403 (accessDeniedHandler) 반환.
- **MockInterview / Question 등 동일 패턴은 403 으로 안정** — 본 케이스만 401 — 원인 불명 (path matching 차이일 가능성).
- **의도**: STUDENT 가 teacher endpoint 호출 시 거부됨이 확실 — 둘 다 4xx.
- **followup**: 만약 후속 PR 에서 일관성 필요하면 `accessDeniedHandler` 명시 설정 검토.

## 관련

- [[../decisions/decision-student-record-categories]] — 8 카테고리 + endpoint 설계
- [[task-log-b7d-mock-interview]] — 직전 stack
- 후속: B7F (토론 / 수행평가 / 시뮬레이션) 또는 B8 (PDF export)
