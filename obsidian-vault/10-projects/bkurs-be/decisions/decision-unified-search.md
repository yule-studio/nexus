---
title: "B10 결정 — 통합 검색 5 도메인 LIKE aggregator"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T19:55:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, search, b10]
related:
  - "[[../bkurs-be]]"
  - "[[decision-record-bridge]]"
home_hub: bkurs-be
---

# B10 결정 — 통합 검색

## 결론

1. **5 도메인 (StudentRecord/Reading/Discussion/PE/Sim) LIKE aggregator** — JPA query methods.
2. **SearchResult DTO 통합** — sourceType + sourceId + title + snippet + createdAt.
3. **3 endpoint** — TEACHER(학생별)/STUDENT(본인)/OWNER(학원 학생).
4. **LIKE '%q%' (no FULLTEXT)** — PoC 단순, 후속 PR FULLTEXT + ngram.
5. **pagination 없음 (limit 50 default)** — 후속 cursor pagination.

## 왜 필요한가

- 학생부 / 독서 / 토론 등 분산 데이터 → 강사가 학생 자료 한꺼번에 검색.
- 키워드 ("데미안") 로 ReadingNote + bridge 로 생성된 StudentRecord SUBJECT 둘 다 매치.
- 학원장은 학원 학생 별 검색 (생기부 자료 작성 시 인용).

## 4 대안 — 검색 백엔드

### A. JPA LIKE — **채택 (1차)**

- 장점: 의존성 없음. 즉시 구현.
- 단점: full table scan, 한글 tokenize 없음.
- **채택**: PoC 단순.

### B. MySQL FULLTEXT + ngram tokenizer

- 장점: 한글 검색 + 인덱스.
- 단점: ALTER TABLE FULLTEXT 5개 + ngram 설정.
- **기각** (1차): 후속 PR.

### C. Elasticsearch / OpenSearch

- **기각**: 외부 인프라 (보류 항목).

### D. Postgres tsvector

- **기각**: DB 변경 — MySQL 9.6 유지.

## 4 대안 — endpoint 권한

### A. 3 endpoint (TEACHER/STUDENT/OWNER) — **채택**

- 장점: B7E StudentRecord 패턴 일관.

### B. 단일 endpoint + role 분기

- **기각**: service 로직에 권한 검증 섞임.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| GET | /api/v1/teachers/me/students/{sid}/search?q= | TEACHER | enrolled |
| GET | /api/v1/students/me/search?q= | STUDENT | 본인 |
| GET | /api/v1/academies/me/students/{sid}/search?q= | OWNER | 본인 학원 학생 |

## SearchResult 형식

```json
{
  "sourceType": "STUDENT_RECORD",
  "sourceId": 42,
  "title": "[수행평가] 중간고사",
  "snippet": "...95점 우수...",
  "createdAt": "2026-05-20T..."
}
```

`sourceType` enum: STUDENT_RECORD / READING_NOTE / DISCUSSION_POST / PE_SUBMISSION / SIM_RESPONSE.

## 회귀 10

1. 강사 학생 검색 — 5 도메인 매치 합산
2. 학생 본인 검색
3. 학원장 검색 (enrolled 학생)
4. q 비어있음 → 빈 결과
5. STUDENT_RECORD 만 매치 (다른 도메인 키워드 X)
6. ReadingNote book_title 매치
7. Discussion content 매치 (post)
8. PE submission content 매치
9. SIM response decisions+justification 매치
10. 다른 학원 학생 검색 → 403 (TEACHER not enrolled)

## 관련

- [[decision-record-bridge]] — bridge record 도 검색 결과 포함 (StudentRecord)
- 후속: FULLTEXT + ngram / pagination / facet / 정렬 옵션
