---
title: "bkurs-fe api-client — pastQuestion"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, pastQuestion]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/admission]]"
home_hub: "[[../../bkurs-fe]]"
---

# pastQuestion client

> 입시 기출 — MASTER 발행 + 학생 view. BE [[../../../bkurs-be/handover/domains/admission|admission]] 도메인과 매핑.

## 1. 파일

- `lib/api/pastQuestion.ts` — axios 클라이언트
- `lib/hooks/usePastQuestion.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `pastQuestionApi.masterList()` | `GET /v1/master/past-questions` |
| `pastQuestionApi.masterCreate(req)` | `POST /v1/master/past-questions` |
| `pastQuestionApi.masterUpdate(id, req)` | `PATCH /v1/master/past-questions/{id}` |
| `pastQuestionApi.masterDelete(id)` | `DELETE /v1/master/past-questions/{id}` |
| `pastQuestionApi.studentList()` | `GET /v1/students/me/past-questions` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useMasterPastQuestions / useCreatePastQuestion / useUpdatePastQuestion / useDeletePastQuestion / useMyPastQuestions
```

query key 컨벤션: `['pastQuestion', ...sub]` (예: `['pastQuestion', 'mine']`, `['pastQuestion', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/past
- middle-student/past
- master/questions (Past 탭)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/admission]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
