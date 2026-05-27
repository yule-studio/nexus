---
title: "bkurs-fe api-client — majorQuestion"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, majorQuestion]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/admission]]"
home_hub: "[[../../bkurs-fe]]"
---

# majorQuestion client

> 입시 학과 면접 문항 — MASTER 발행 + 학생 view. BE [[../../../bkurs-be/handover/domains/admission|admission]] 도메인과 매핑.

## 1. 파일

- `lib/api/majorQuestion.ts` — axios 클라이언트
- `lib/hooks/useMajorQuestion.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `majorQuestionApi.masterList(grade?)` | `GET /v1/master/major-questions` |
| `majorQuestionApi.masterCreate(req)` | `POST /v1/master/major-questions` |
| `majorQuestionApi.masterUpdate(id, req)` | `PATCH /v1/master/major-questions/{id}` |
| `majorQuestionApi.masterDelete(id)` | `DELETE /v1/master/major-questions/{id}` |
| `majorQuestionApi.studentList()` | `GET /v1/students/me/major-questions` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useMasterMajorQuestions / useCreateMajorQuestion / useUpdateMajorQuestion / useDeleteMajorQuestion / useMyMajorQuestions
```

query key 컨벤션: `['majorQuestion', ...sub]` (예: `['majorQuestion', 'mine']`, `['majorQuestion', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/major
- master/questions (Major 탭)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/admission]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
