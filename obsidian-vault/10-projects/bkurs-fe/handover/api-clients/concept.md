---
title: "bkurs-fe api-client — concept"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, concept]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/concept]]"
home_hub: "[[../../bkurs-fe]]"
---

# concept client

> 진로 / 컨셉 진단 (중·고등 공용). BE [[../../../bkurs-be/handover/domains/concept|concept]] 도메인과 매핑.

## 1. 파일

- `lib/api/concept.ts` — axios 클라이언트
- `lib/hooks/useConcept.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `conceptApi.getMine()` | `GET /v1/students/me/concept-diagnosis` |
| `conceptApi.updateProgress(req)` | `PUT /v1/students/me/concept-diagnosis/progress` |
| `conceptApi.complete(req)` | `POST /v1/students/me/concept-diagnosis/complete` |
| `conceptApi.byTeacher(sid)` | `GET /v1/teachers/me/students/{id}/concept-diagnosis` |
| `conceptApi.approve(id)` | `POST /v1/teachers/me/concept-diagnoses/{id}/approve` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useConceptDiagnosis.useGetMine / useUpdateProgress / useComplete / useByTeacher / useApprove
```

query key 컨벤션: `['concept', ...sub]` (예: `['concept', 'mine']`, `['concept', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/concept
- middle-student/concept

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/concept]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
