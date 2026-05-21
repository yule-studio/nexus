---
title: "bkurs-fe api-client — auth"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, auth]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/auth]]"
home_hub: "[[../../bkurs-fe]]"
---

# auth client

> 로그인 / signup / refresh / logout / me / changePassword. BE [[../../../bkurs-be/handover/domains/auth|auth]] 도메인과 매핑.

## 1. 파일

- `lib/api/auth.ts` — axios 클라이언트
- `lib/hooks/useAuth.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `authApi.login(email, password)` | `POST /api/v1/auth/login` |
| `authApi.signup(email, password, role)` | `POST /api/v1/auth/signup` |
| `authApi.refresh()` | `POST /api/v1/auth/refresh (HttpOnly cookie)` |
| `authApi.logout()` | `POST /api/v1/auth/logout` |
| `authApi.me()` | `GET /api/v1/auth/me` |
| `authApi.changePassword(old, new)` | `POST /api/v1/auth/change-password` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useLogin / useMe / useChangePassword / (refresh interceptor)
```

query key 컨벤션: `['auth', ...sub]` (예: `['auth', 'mine']`, `['auth', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- 모든 role 의 login 페이지
- _shared/auth/useStudent*

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/auth]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
