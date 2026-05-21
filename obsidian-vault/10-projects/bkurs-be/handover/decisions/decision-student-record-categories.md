---
title: "B7E 결정 — StudentRecord (생기부) 8 카테고리 + 강사 입력"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T15:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, student-record, b7e]
related:
  - "[[../bkurs-be]]"
home_hub: bkurs-be
---

# B7E 결정 — StudentRecord (생기부)

## 결론

1. **`student_record_entry` 단일 테이블 + `category` enum (8종)** — 학생부 영역 통합.
2. **카테고리**: AWARD (수상) / AUTONOMY (자율활동) / CLUB (동아리) / CAREER (진로) / SERVICE (봉사) / SUBJECT (교과세특) / BEHAVIOR (행동특성) / READING (독서).
3. **강사가 본인 수강생 entry 입력** — period (학기 표시) + title + content + author audit.
4. **학생 본인 + 학원장 본인 학원** 조회 가능 (카테고리 필터).

## 왜 필요한가

- 생기부 = 학종 핵심 산출물. 학원이 학기 동안 입력한 내용이 대학 자료로.
- 카테고리별 누적이 학생부 표준 (교육부 학종 기준).
- 강사가 수업 중 관찰한 내용을 학기 종료 시 일괄 정리.

## 4 대안 비교 — 모델

### A. 카테고리별 별도 테이블 — 기각

- 단점: 8 테이블 × 동일 CRUD = 코드 중복 폭증.
- **기각**.

### B. 단일 entry + category enum — **채택**

- 장점: 단일 schema/service. 카테고리 필터로 분기.
- **채택**: 운영 / 확장 모두 단순.

### C. JSON 컬럼 (entries_json)

- **기각**: 검색/필터링 불가.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/teachers/me/students/{studentId}/records | TEACHER | 본인 수강생 |
| GET | /api/v1/teachers/me/students/{studentId}/records | TEACHER | 본인 수강생 |
| PATCH | /api/v1/teachers/me/records/{id} | TEACHER | 본인 작성 |
| DELETE | /api/v1/teachers/me/records/{id} | TEACHER | 본인 작성 |
| GET | /api/v1/students/me/records?category= | STUDENT | 본인 |
| GET | /api/v1/academies/me/students/{studentId}/records?category= | OWNER | 본인 학원 학생 |

## 회귀 라인 (10)

1. 강사 AWARD 입력 → 201
2. 강사 CLUB 입력 (다른 카테고리)
3. STUDENT 입력 403
4. 강사 본인 수강생 조회
5. category=AWARD 필터
6. 본인 외 수강생 → 403
7. 학생 본인 record 전체 조회
8. 학원장 본인 학원 학생 조회
9. 강사 PATCH 본인 작성 만
10. DELETE → CASCADE 없음 (단일 테이블)

## 관련

- [[decision-question-attempt-model]] — type enum 패턴 재사용
- 후속: B8 PDF export
