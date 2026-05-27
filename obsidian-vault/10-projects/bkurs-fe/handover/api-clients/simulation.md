---
title: "bkurs-fe api-client — simulation"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, simulation]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/simulation]]"
home_hub: "[[../../bkurs-fe]]"
---

# simulation client

> 모의지원 (3 type) + 재제출 review 무효화 (BE bridge → record CAREER). BE [[../../../bkurs-be/handover/domains/simulation|simulation]] 도메인과 매핑.

## 1. 파일

- `lib/api/simulation.ts` — axios 클라이언트
- `lib/hooks/useSimulation.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `simulationApi.listMine()` | `GET /v1/students/me/simulation-cases` |
| `simulationApi.respond(caseId, answer)` | `POST /v1/students/me/simulation-cases/{id}/respond` |
| `simulationApi.createByCourse(courseId, req)` | `POST /v1/teachers/me/courses/{id}/simulation-cases` |
| `simulationApi.listResponses(id)` | `GET /v1/teachers/me/simulation-cases/{id}/responses` |
| `simulationApi.review(responseId, req)` | `POST /v1/teachers/me/simulation-responses/{id}/review` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useSimulation.useMine / useRespond / useCreate / useListResponses / useReview
```

query key 컨벤션: `['simulation', ...sub]` (예: `['simulation', 'mine']`, `['simulation', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/simulation
- middle-student/simulation

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/simulation]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
