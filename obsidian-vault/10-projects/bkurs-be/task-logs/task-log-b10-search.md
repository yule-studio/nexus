---
title: "B10 task-log — 통합 검색 5 도메인"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T19:55:00+09:00
agent: engineering-agent/coding
tags: [task-log, search, b10]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-unified-search]]"
home_hub: bkurs-be
---

# B10 task-log — 통합 검색

## 결과

- 이슈: #53
- PR: #54 (draft, stacked on #52 B9)
- 브랜치: `feature/issue-53-b10-search`
- 4 commit (S1~S4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| 93b2fad | 5 repo 검색 메서드 + SearchResult DTO | +52 | ContainingIgnoreCase |
| 79d3fd8 | SearchService aggregator | +123 | 5 도메인 + 권한 3 |
| 28c3adf | 3 controller + handler | +106 | TEACHER/STUDENT/OWNER |
| 0452bf0 | SearchApiTest 10 + CHANGELOG | +272 | 5 도메인 cross-match |

## 3 endpoint

- TEACHER — GET `/api/v1/teachers/me/students/{sid}/search?q=`
- STUDENT — GET `/api/v1/students/me/search?q=`
- OWNER — GET `/api/v1/academies/me/students/{sid}/search?q=`

## 5 SearchResultType

| enum | 매치 필드 | source |
|---|---|---|
| STUDENT_RECORD | title + content | StudentRecordEntry (수동 + bridge) |
| READING_NOTE | bookTitle + impression | ReadingNote |
| DISCUSSION_POST | content | DiscussionPost (학생 발화) |
| PE_SUBMISSION | content | PerformanceSubmission |
| SIM_RESPONSE | decisions + justification | SimulationResponse |

## 10 회귀 (SearchApiTest)

1. 강사 학생 검색 — 5 도메인 cross-match ("데미안" 키워드) ✅
2. 학생 본인 검색 ✅
3. 학원장 검색 (enrolled 학생) ✅
4. q 빈 → 빈 결과 ✅
5. STUDENT_RECORD title 매치 ✅
6. ReadingNote impression 매치 ✅
7. DiscussionPost content 매치 ✅
8. PE submission content 매치 ✅
9. Sim response decisions/justification 매치 ✅
10. 다른 학원 학생 → 403 ✅

## 선택 이유 — handover

### A. JPA LIKE (FULLTEXT 보류)

- **선택 이유**: 의존성 없음, 즉시 구현, PoC 단순.
- **기각된 대안**: FULLTEXT + ngram (ALTER TABLE 5개 + 설정 부담).
- **trade-off**: full table scan. 데이터 누적 시 후속 FULLTEXT 전환 필요.

### B. SearchResult 통합 DTO (도메인별 DTO 합산 X)

- **선택 이유**: 클라이언트 UI 가 통합 결과 (정렬/필터) 처리 용이.
- **기각된 대안**: 5 도메인 별 응답 (5 호출 + 클라이언트 merge).
- **trade-off**: 도메인 별 상세는 클라이언트가 sourceId 로 별도 조회.

### C. 학생 본인 발화/제출만 검색 (강사/학원장 작성 record 제외)

- **선택 이유**: 검색 의도 = "학생 자료" — 강사 평가/feedback 은 record 에 통합되어 검색됨 (bridge).
- **기각된 대안**: 강사 작성 ReadingFeedback/PE.feedback 도 검색 — 의미 분리 흐려짐.

### D. limit 50 default, pagination 없음

- **선택 이유**: PoC 단순. 5 도메인 통합 결과 50건 = 일반 학생 충분.
- **기각된 대안**: cursor pagination — 후속 PR.
- **future**: createdAt cursor 또는 page 기반.

### E. 권한 별 3 endpoint (단일 endpoint role 분기 X)

- **선택 이유**: B7E StudentRecord 패턴 일관 — 권한 명확.
- **기각된 대안**: 단일 /search?q=&studentId= + role 분기 (logic 섞임).

## 관련

- [[../decisions/decision-unified-search]]
- [[task-log-b9-record-bridge]] — bridge record 도 검색 결과 포함
- 후속: FULLTEXT + ngram / pagination / facet / B11 통계
