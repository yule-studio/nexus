---
title: "bkurs-fe api-client — interview"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, interview]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/interview]]"
home_hub: "[[../../bkurs-fe]]"
---

# interview client

> 모의면접 (4 type) + 미디어 응답 (audio/video URL) + 평가. BE [[../../../bkurs-be/handover/domains/interview|interview]] 도메인과 매핑.

## 1. 파일

- `lib/api/interview.ts` — axios 클라이언트
- `lib/hooks/useInterview.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `interviewApi.listMine()` | `GET /v1/students/me/interviews` |
| `interviewApi.respond(id, req)` | `POST /v1/students/me/interviews/{id}/respond` |
| `interviewApi.myResponses()` | `GET /v1/students/me/interview-responses` |
| `interviewApi.createByCourse(courseId, req)` | `POST /v1/teachers/me/courses/{id}/interviews` |
| `interviewApi.update(id, req)` | `PATCH /v1/teachers/me/interviews/{id}` |
| `interviewApi.evaluate(responseId, req)` | `POST /v1/teachers/me/responses/{id}/evaluate` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useInterview.useMine / useRespond / useMyResponses / useCreate / useUpdate / useEvaluate
```

query key 컨벤션: `['interview', ...sub]` (예: `['interview', 'mine']`, `['interview', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/mockexam (후속)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/interview]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
