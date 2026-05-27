---
title: "도메인 — reading (독서 노트 + 강사 feedback)"
kind: handover
status: stable
created_at: 2026-05-21T23:53:00+09:00
tags: [bkurs-be, handover, domain, reading]
related:
  - "[[../../bkurs-be]]"
  - "[[record]]"
  - "[[../decisions/decision-reading-note-model]]"
home_hub: "[[../../bkurs-be]]"
---

# reading 도메인

> 학생이 자유롭게 작성하는 독서 노트. **course / academy 격리 없음** — 학원 외 활동도 포함.
> 강사가 노트별 feedback (uk + recommended_score 0~5). **record bridge → READING 카테고리**.

## 1. 책임

- 학생 ReadingNote CRUD (본인 owner)
- 강사 (학생을 가르치는 강사) feedback (uk + recommended_score 0~5) — `feedback()` 시 record bridge
- academy 격리 없음 — 다른 학원 이동해도 본인 노트 유지

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `ReadingNote` | id / studentAccountId / bookTitle / author / pagesRead / impressions / createdAt — **course/academy FK 없음** |
| `ReadingFeedback` | uk(note_id, teacher_account_id) + comment / recommendedScore 0~5 / createdAt |

## 3. 패키지 구성

```
reading/
├── controller/
│   ├── StudentReadingController     # /api/v1/students/me/reading-notes — CRUD
│   └── TeacherReadingController     # /api/v1/teachers/me — 학생 노트 조회 + feedback
├── service/                         # ReadingService — CRUD + feedback upsert + event publish
├── domain/
├── dto/{request,response}/
├── enums/                           # ReadingErrorCode (RDN-)
├── exception/
└── handler/
```

## 4. API endpoint

### STUDENT (5)

| Method | Path | 책임 |
| --- | --- | --- |
| POST/GET | `/api/v1/students/me/reading-notes` | 본인 노트 CRUD |
| PATCH/DELETE | `/api/v1/students/me/reading-notes/{id}` | |
| GET | `/api/v1/students/me/reading-notes/{id}/feedback` | 받은 feedback 목록 |

### TEACHER (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/teachers/me/students/{studentId}/reading-notes` | 학생 노트 목록 |
| POST | `/api/v1/teachers/me/reading-notes/{noteId}/feedback` | feedback upsert (uk) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V20 | `reading_note` + `reading_feedback` (uk + score 0~5) |

## 6. 핵심 코드 흐름

### feedback (upsert + bridge)

```
ReadingService.feedback(noteId, teacherId, comment, score)
  ▼
1. note = readingNoteRepo.findById(noteId)
2. 강사 자격 검증 — 본인 수업에 enrolled 학생인지 (CourseEnrollmentRepository)
3. existing = feedbackRepo.findByNoteIdAndTeacherAccountId(...)
     ├── 있음 → existing.update(comment, score, now)
     └── 없음 → save
4. eventPublisher.publishEvent(new ReadingFeedbackedEvent(
       studentId=note.studentAccountId, noteId=note.id, content=note.bookTitle + " ..."))
       ─→ record bridge: READING 카테고리 upsert
```

### academy 격리 없음 — 강사 권한 검증

학생이 학원을 이동해도 노트는 유지. 강사 입장에서는:

- 본인 수업에 **현재 enrolled 인 학생** 의 노트만 조회 / feedback.
- 이전 학원에서 작성한 노트도 보임 (좋은 의도) — privacy 정책상 OK.

## 7. 의존

- **의존**: auth, course (강사 enrollment 검증), record (event publish)
- **의존받음**: record bridge, stats (독서 평균 score), search (도서명 검색)

## 서비스 로직 (`reading/service/`)

1 service. **academy 격리 없음** (학생이 학원 이동해도 노트 유지).

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `ReadingService` | `@Transactional` (write) / `readOnly` (조회) | 학생 노트 CRUD + 강사 feedback upsert + READING bridge event |

### 핵심 메서드

```java
public ReadingNote createByStudent(Long studentAccountId, String bookTitle, String author, Integer pagesRead, String impressions)
public ReadingNote updateByStudent(Long studentAccountId, Long noteId, ...)
public void deleteByStudent(Long studentAccountId, Long noteId)
public List<ReadingNote> listByStudent(Long studentAccountId)
public List<ReadingNote> listByTeacherForStudent(Long teacherAccountId, Long studentAccountId)
                                                          // 강사 본인 수업에 현재 ACTIVE enrolled 학생인지 검증

public ReadingFeedback submitFeedbackByTeacher(Long teacherAccountId, Long noteId, String comment, Integer recommendedScore)
                                                          // uk(note_id, teacher_account_id) upsert + score 0~5
                                                          // → publishEvent(new ReadingFeedbackedEvent(...))
                                                          //   record bridge 가 READING 카테고리로 upsert (same TX)
public List<ReadingFeedback> listFeedbackForStudent(Long studentAccountId, Long noteId)
public List<ReadingFeedback> listFeedbackForTeacher(Long teacherAccountId, Long noteId)
```

## cross-link

- [[record]] — READING bridge source
- [[../decisions/decision-reading-note-model]]
