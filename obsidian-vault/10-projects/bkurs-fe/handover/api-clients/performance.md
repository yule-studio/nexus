---
title: "bkurs-fe api-client — performance"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, performance]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/performance]]"
home_hub: "[[../../bkurs-fe]]"
---

# performance client

> 수행평가 + 재제출 채점 무효화 (BE bridge → record SUBJECT). BE [[../../../bkurs-be/handover/domains/performance|performance]] 도메인과 매핑.

## 1. 파일

- `lib/api/performance.ts` — axios 클라이언트
- `lib/hooks/usePerformance.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `performanceApi.listMine()` | `GET /v1/students/me/performance-evaluations` |
| `performanceApi.submit(evalId, content)` | `POST /v1/students/me/performance-evaluations/{id}/submit` |
| `performanceApi.createByCourse(courseId, req)` | `POST /v1/teachers/me/courses/{id}/performance-evaluations` |
| `performanceApi.listSubmissions(id)` | `GET /v1/teachers/me/performance-evaluations/{id}/submissions` |
| `performanceApi.grade(submissionId, req)` | `POST /v1/teachers/me/performance-submissions/{id}/grade` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
usePerformance.useMine / useSubmit / useCreate / useListSubmissions / useGrade
```

query key 컨벤션: `['performance', ...sub]` (예: `['performance', 'mine']`, `['performance', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- (admin 학생 상세 후속)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/performance]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
