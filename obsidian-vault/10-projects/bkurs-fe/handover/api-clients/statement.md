---
title: "bkurs-fe api-client — statement"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, statement]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/statement]]"
home_hub: "[[../../bkurs-fe]]"
---

# statement client

> 자소서 DRAFT/SUBMITTED + 강사 feedback. BE [[../../../bkurs-be/handover/domains/statement|statement]] 도메인과 매핑.

## 1. 파일

- `lib/api/statement.ts` — axios 클라이언트
- `lib/hooks/useStatement.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `statementApi.listMine()` | `GET /v1/students/me/personal-statements` |
| `statementApi.createMine(req)` | `POST /v1/students/me/personal-statements` |
| `statementApi.updateMine(id, req)` | `PATCH /v1/students/me/personal-statements/{id}` |
| `statementApi.submitMine(id)` | `POST /v1/students/me/personal-statements/{id}/submit` |
| `statementApi.listForFeedback(id)` | `GET /v1/students/me/personal-statements/{id}/feedback` |
| `statementApi.listByTeacher(sid)` | `GET /v1/teachers/me/students/{id}/personal-statements` |
| `statementApi.getByTeacher(id)` | `GET /v1/teachers/me/personal-statements/{id}` |
| `statementApi.addFeedback(id, content)` | `POST /v1/teachers/me/personal-statements/{id}/feedback` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useStatement.useMine / useCreateMine / useUpdateMine / useSubmitMine / useFeedback / useByTeacher
```

query key 컨벤션: `['statement', ...sub]` (예: `['statement', 'mine']`, `['statement', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- (후속 — 자소서 화면)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/statement]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
