---
title: "bkurs-fe api-client — question"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, question]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/question]]"
home_hub: "[[../../bkurs-fe]]"
---

# question client

> 강사 발행 예상/기출 (EXPECTED/PAST) + 학생 attempt upsert. BE [[../../../bkurs-be/handover/domains/question|question]] 도메인과 매핑.

## 1. 파일

- `lib/api/question.ts` — axios 클라이언트
- `lib/hooks/useQuestion.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `questionApi.listMine(type?)` | `GET /v1/students/me/questions` |
| `questionApi.attempt(qid, content)` | `POST /v1/students/me/questions/{id}/attempt` |
| `questionApi.myAttempts()` | `GET /v1/students/me/attempts` |
| `questionApi.createByCourse(courseId, req)` | `POST /v1/teachers/me/courses/{id}/questions` |
| `questionApi.update(id, req)` | `PATCH /v1/teachers/me/questions/{id}` |
| `questionApi.score(attemptId, req)` | `POST /v1/teachers/me/attempts/{id}/score` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useQuestion.useMine / useAttempt / useMyAttempts / useCreateByCourse / useUpdate / useScore
```

query key 컨벤션: `['question', ...sub]` (예: `['question', 'mine']`, `['question', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/expect
- middle-student/expect

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/question]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
