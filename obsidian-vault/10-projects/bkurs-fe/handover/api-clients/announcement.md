---
title: "bkurs-fe api-client — announcement"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, announcement]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/announcement]]"
home_hub: "[[../../bkurs-fe]]"
---

# announcement client

> 학원 공지 CRUD + 읽음 mark. BE [[../../../bkurs-be/handover/domains/announcement|announcement]] 도메인과 매핑.

## 1. 파일

- `lib/api/announcement.ts` — axios 클라이언트
- `lib/hooks/useAnnouncement.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `announcementApi.listByOwner()` | `GET /v1/academies/me/announcements` |
| `announcementApi.listForMe()` | `GET /v1/academies/my/announcements` |
| `announcementApi.create(req)` | `POST /v1/academies/me/announcements` |
| `announcementApi.update(id, req)` | `PATCH /v1/academies/me/announcements/{id}` |
| `announcementApi.delete(id)` | `DELETE /v1/academies/me/announcements/{id}` |
| `announcementApi.markRead(id)` | `POST /v1/academies/my/announcements/{id}/read` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useAnnouncement.useOwner / useForMe / useCreate / useUpdate / useDelete / useMarkRead
```

query key 컨벤션: `['announcement', ...sub]` (예: `['announcement', 'mine']`, `['announcement', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- admin/announcements
- 각 role 사이드바 알림

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/announcement]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
