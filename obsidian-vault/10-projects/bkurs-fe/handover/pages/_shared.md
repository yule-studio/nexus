---
title: "bkurs-fe pages — _shared (role 간 공용)"
kind: handover
status: stable
created_at: 2026-05-22T01:29:00+09:00
tags: [bkurs-fe, handover, pages, shared, auth]
related:
  - "[[catalog]]"
  - "[[high-student]]"
  - "[[middle-student]]"
home_hub: "[[../../bkurs-fe]]"
---

# _shared (role 간 공용) 페이지

> 2+ role 에서 사용하는 page-level helper. 현재 학생 (high + middle) 공용 인증 4 hook.

## 1. 디렉토리

```
pages/_shared/
└── auth/
    ├── useStudentLogin.ts
    ├── useStudentSignup.ts
    ├── useStudentConnect.ts
    └── useStudentPending.ts
```

> 컴포넌트 없음 — 4 hook 만. high + middle 양쪽 login/signup/connect/pending 화면이 동일 hook 을
> `group: 'high' | 'middle'` 인자로 호출.

## 2. 4 hook 의 책임

### 2.1 `useStudentLogin(group)`

```ts
export type StudentGroup = 'high' | 'middle'
```

흐름:

```
1. authApi.login({ email, password })
2. role 검증 — 응답 role 이 STUDENT_HIGH (group='high') 또는 STUDENT_MIDDLE 인지
   - 불일치 → "고등/중등 계정만 가능" 에러
3. status 별 navigate 분기:
   - PENDING_CONNECT → /<group>-student/connect (학원 코드 입력)
   - PENDING_APPROVE → /<group>-student/pending (학원장 승인 대기)
   - ACTIVE          → /<group>-student/roadmap
   - REJECTED        → 에러 메시지 (login 화면 유지)
4. studentTokenAtom + studentProfileAtom 갱신 (atomWithStorage → localStorage 자동)
```

### 2.2 `useStudentSignup(group)`

```ts
authApi.signup({ email, password, role: GROUP_TO_ROLE[group] })
// 응답 status=PENDING_CONNECT → navigate /<group>-student/connect
```

### 2.3 `useStudentConnect(group)`

학원 코드 입력 → `POST /api/v1/auth/connect`. membership row 생성 (status=PENDING_APPROVE) →
account status=PENDING_APPROVE 갱신 → navigate `/<group>-student/pending`.

### 2.4 `useStudentPending(group)`

승인 대기 화면 hook. polling 또는 사용자 trigger refresh:
- `GET /api/v1/auth/me` 호출
- status=ACTIVE 되면 navigate `/<group>-student/roadmap`
- status=REJECTED 되면 에러 메시지

## 3. 4 status 매핑 (FE ↔ BE)

| FE StudentStatus (lib/auth/studentAuth) | BE AccountStatus (auth/domain/) | UI |
| --- | --- | --- |
| `pending` (PENDING_CONNECT) | `PENDING_CONNECT` | `/connect` |
| `pending` (PENDING_APPROVE) | `PENDING_APPROVE` | `/pending` |
| `active` | `ACTIVE` | 정상 (`/roadmap`) |
| `rejected` | `REJECTED` 또는 BLOCKED | login 에러 |

자세히 → [[../architecture/state-and-auth]] §4

## 4. role 간 공유 vs 한정 — 결정 기준

| 위치 | 의미 |
| --- | --- |
| `pages/{role}/_components/` | 본 role 내 1+ 페이지에서 재사용. cross-role 사용 없음 |
| `pages/_shared/` | 2+ role 에서 사용하는 page-level helper (현재 auth 4 hook 만) |
| `components/` (top) | role 무관 layout (`RootLayout`) |
| `lib/` | 비-React logic (api / hooks / atoms / util) |

> ThreadView 가 high 와 middle 양쪽 `_components/` 에 별도 — 디자인 variation (brand 색 / 톤). 통일
> 시 `_shared/ThreadView.tsx` 로 승격 후보.

## 5. cross-link

- [[high-student]] / [[middle-student]] — 4 hook caller
- [[../architecture/routing-and-protection]] — 학생 4 status 분기 가드
- [[../architecture/state-and-auth]] — studentProfileAtom + 4 status 흐름
- [[../api-clients/auth]] — authApi.{login,signup,me}
- [[../api-clients/academy]] — `/api/v1/auth/connect`
