---
title: "bkurs-fe api-client — homework"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, homework]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/homework]]"
home_hub: "[[../../bkurs-fe]]"
---

# homework client

> 숙제 + 학생 제출 (upsert) + 강사 채점 (0~100). BE [[../../../bkurs-be/handover/domains/homework|homework]] 도메인과 매핑.

## 1. 파일

- `lib/api/homework.ts` — axios 클라이언트
- `lib/hooks/useHomework.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `homeworkApi.listMine()` | `GET /v1/students/me/homework` |
| `homeworkApi.submit(homeworkId, content)` | `POST /v1/students/me/homework/{id}/submit` |
| `homeworkApi.mySubmissions()` | `GET /v1/students/me/submissions` |
| `homeworkApi.createForCourse(courseId, req)` | `POST /v1/teachers/me/courses/{id}/homework` |
| `homeworkApi.listByCourse(courseId)` | `GET /v1/teachers/me/courses/{id}/homework` |
| `homeworkApi.update(id, req)` | `PATCH /v1/teachers/me/homework/{id}` |
| `homeworkApi.delete(id)` | `DELETE /v1/teachers/me/homework/{id}` |
| `homeworkApi.listSubmissions(id)` | `GET /v1/teachers/me/homework/{id}/submissions` |
| `homeworkApi.grade(submissionId, req)` | `POST /v1/teachers/me/submissions/{id}/grade` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useMyHomework / useHomeworkByCourse / useSubmitHomework / useMySubmissions / useCreateHomework / useUpdateHomework / useDeleteHomework / useHomeworkSubmissions / useGradeSubmission
```

query key 컨벤션: `['homework', ...sub]` (예: `['homework', 'mine']`, `['homework', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- middle-student/homework
- admin/students/detail/middle-tabs/middle-homework

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/homework]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
