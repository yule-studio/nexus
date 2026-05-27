---
title: "bkurs-fe api-client — record"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, record]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/record]]"
home_hub: "[[../../bkurs-fe]]"
---

# record client

> 학생부 8 카테고리 (수동 + 4 도메인 자동 bridge). BE [[../../../bkurs-be/handover/domains/record|record]] 도메인과 매핑.

## 1. 파일

- `lib/api/record.ts` — axios 클라이언트
- `lib/hooks/useRecord.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `recordApi.listMine(category?)` | `GET /v1/students/me/records` |
| `recordApi.createByTeacher(sid, req)` | `POST /v1/teachers/me/students/{id}/records` |
| `recordApi.listByTeacher(sid, category?)` | `GET /v1/teachers/me/students/{id}/records` |
| `recordApi.update(id, content)` | `PATCH /v1/teachers/me/records/{id}` |
| `recordApi.delete(id)` | `DELETE /v1/teachers/me/records/{id}` |
| `recordApi.listByOwner(sid, category?)` | `GET /v1/academies/me/students/{id}/records` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useRecord.useMine / useCreateByTeacher / useByTeacher / useUpdate / useDelete / useByOwner
```

query key 컨벤션: `['record', ...sub]` (예: `['record', 'mine']`, `['record', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/record
- admin/students/detail (record 탭)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/record]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
