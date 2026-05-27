---
title: "도메인 — search (5 도메인 통합 LIKE 검색)"
kind: handover
status: stable
created_at: 2026-05-21T23:56:00+09:00
tags: [bkurs-be, handover, domain, search]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-unified-search]]"
home_hub: "[[../../bkurs-be]]"
---

# search 도메인

> 5 도메인 (Record / Reading / Discussion / PerformanceSubmission / SimulationResponse) 의
> 자유 텍스트 컬럼에 대한 `LIKE '%q%'` 통합 검색 + role 분기. **엔티티 / migration 없음** — 순수
> aggregator.

## 1. 책임

- 5 도메인 repository 의 `findBy...ContainingIgnoreCase` 메서드 호출
- 결과 통합 → `SearchResult` record (snippet 100자) — createdAt desc + limit 50
- role 별 endpoint 3개 (학생 본인 / 강사 학생 / 학원장 학생)

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `SearchResultType` (enum) | RECORD / READING / DISCUSSION / PERFORMANCE / SIMULATION |
| `SearchResult` (record) | type / sourceId / snippet (100자) / createdAt |

## 3. 패키지 구성

```
search/
├── controller/
│   ├── TeacherSearchController      # /api/v1/teachers/me/students/{id}/search
│   ├── StudentSearchController      # /api/v1/students/me/search
│   └── OwnerSearchController        # /api/v1/academies/me/students/{id}/search
├── service/                         # SearchService — aggregator
├── dto/{request,response}/          # SearchQueryRequest, SearchResult
├── enums/                           # SearchErrorCode (SCH-)
├── exception/
└── handler/
```

## 4. API endpoint

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| GET | `/api/v1/students/me/search?q=...` | STUDENT_* | 본인 데이터 검색 |
| GET | `/api/v1/teachers/me/students/{studentId}/search?q=...` | TEACHER | 본인 수업 학생 검색 |
| GET | `/api/v1/academies/me/students/{studentId}/search?q=...` | OWNER | 학원 학생 검색 |

## 5. 핵심 코드 흐름

```
SearchService.searchForStudent(studentAccountId, q)
  ▼
List<SearchResult> result = new ArrayList<>();
result.addAll(recordRepo.findByStudentAccountIdAndContentContainingIgnoreCase(sid, q)
                .stream().map(e -> SearchResult.of(RECORD, e)).toList());
result.addAll(readingRepo.findByStudentAccountIdAndBookTitleContainingIgnoreCase(...));
result.addAll(discussionPostRepo.findByAccountIdAndBodyContainingIgnoreCase(...));
result.addAll(performanceSubmissionRepo.findByStudentAccountIdAndContentContainingIgnoreCase(...));
result.addAll(simulationResponseRepo.findByStudentAccountIdAndAnswerContainingIgnoreCase(...));

return result.stream()
    .sorted(Comparator.comparing(SearchResult::createdAt).reversed())
    .limit(50)
    .toList();
```

각 repository 에 `*ContainingIgnoreCase` 메서드가 미리 정의돼 있어야 함 (5 도메인 모두).

## 6. 한계 / 후속

- LIKE '%q%' — 인덱스 안 탐. 데이터 100k row 넘어가면 응답 지연.
- 후속: MySQL FULLTEXT + ngram parser (한글 검색) + pagination.
- 5 도메인 hit 가 한 응답에 섞임 — type 별 weight / scoring 없음.

## 7. 의존

- **의존**: record, reading, discussion, performance, simulation (Repository 5개)
- **의존받음**: (현재 없음 — endpoint 직접 사용)

## 서비스 로직 (`search/service/`)

1 service. 엔티티 / Repository 없음 — 5 도메인 repo aggregator + role 분기.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `SearchService` | `@Transactional(readOnly)` | 5 도메인 LIKE 호출 + createdAt desc + limit 50 |

### 핵심 메서드

```java
public List<SearchResult> searchByStudent(Long studentAccountId, String q)
                                                          // 학생 본인 데이터 검색
public List<SearchResult> searchByTeacherForStudent(Long teacherAccountId, Long studentAccountId, String q)
                                                          // 강사 본인 수업 학생 검증 → 학생 데이터 검색
public List<SearchResult> searchByOwnerForStudent(Long ownerAccountId, Long studentAccountId, String q)
                                                          // 학원장 본인 학원 학생 검증 → 학생 데이터 검색
```

내부 흐름 (3 메서드 공통, 권한 검증만 다름):

```java
List<SearchResult> result = new ArrayList<>();
result.addAll(recordRepo.findByStudentAccountIdAndContentContainingIgnoreCase(sid, q).stream().map(...).toList());
result.addAll(readingRepo.findByStudentAccountIdAndBookTitleContainingIgnoreCase(...));
result.addAll(discussionPostRepo.findByAccountIdAndBodyContainingIgnoreCase(...));
result.addAll(performanceSubmissionRepo.findByStudentAccountIdAndContentContainingIgnoreCase(...));
result.addAll(simulationResponseRepo.findByStudentAccountIdAndAnswerContainingIgnoreCase(...));
return result.stream().sorted(Comparator.comparing(SearchResult::createdAt).reversed()).limit(50).toList();
```

후속: FULLTEXT + ngram parser + pagination.

## cross-link

- [[../decisions/decision-unified-search]]
- [[record]] / [[reading]] / [[discussion]] / [[performance]] / [[simulation]]
