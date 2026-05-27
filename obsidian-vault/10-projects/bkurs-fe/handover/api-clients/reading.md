---
title: "bkurs-fe api-client — reading"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, reading]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/reading]]"
home_hub: "[[../../bkurs-fe]]"
---

# reading client

> 독서 노트 (academy 격리 없음) + 강사 feedback (BE bridge → record READING). BE [[../../../bkurs-be/handover/domains/reading|reading]] 도메인과 매핑.

## 1. 파일

- `lib/api/reading.ts` — axios 클라이언트
- `lib/hooks/useReading.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `readingApi.listMine()` | `GET /v1/students/me/reading-notes` |
| `readingApi.createMine(req)` | `POST /v1/students/me/reading-notes` |
| `readingApi.updateMine(id, req)` | `PATCH /v1/students/me/reading-notes/{id}` |
| `readingApi.deleteMine(id)` | `DELETE /v1/students/me/reading-notes/{id}` |
| `readingApi.listFeedbackForStudent(id)` | `GET /v1/students/me/reading-notes/{id}/feedback` |
| `readingApi.listByTeacher(sid)` | `GET /v1/teachers/me/students/{id}/reading-notes` |
| `readingApi.feedbackByTeacher(noteId, req)` | `POST /v1/teachers/me/reading-notes/{id}/feedback` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useReading.useMine / useCreateMine / useUpdateMine / useDeleteMine / useFeedback / useByTeacher / useFeedbackByTeacher
```

query key 컨벤션: `['reading', ...sub]` (예: `['reading', 'mine']`, `['reading', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/book
- middle-student/book

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/reading]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
