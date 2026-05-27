---
title: "bkurs-fe — 라우팅 / 인증 가드"
kind: handover
status: stable
created_at: 2026-05-21T23:05:00+09:00
tags: [bkurs-fe, handover, routing, react-router, auth]
related:
  - "[[bkurs-fe]]"
  - "[[directory-structure]]"
  - "[[role-modules]]"
  - "[[state-and-auth]]"
home_hub: bkurs-fe
---

# bkurs-fe — 라우팅 / 인증 가드

> `src/router.tsx` + 4 role layout 의 `<Navigate>` 패턴 + 학생 4 status 분기. 라우트 추가 / 가드
> 동작 / 외부 진입 시 redirect 어떻게 결정되는가.

## 1. 한 장 요약

```
<BrowserRouter>
  /
    └── <RootLayout>
         └── HomePage (랜딩 — role 선택 또는 자동 redirect)
  /high-student/login           ─── public
  /high-student/signup          ─── public
  /high-student/connect         ─── public (token 있어야 의미)
  /high-student/pending         ─── public (token 있어야 의미)
  /high-student/*               ─── <HighStudentLayout> 가드 + sub-routes 13
  /middle-student/...           ─── 동일 패턴 (15 sub)
  /master/login                 ─── public
  /master/*                     ─── <MasterLayout> 가드 + 9 sub
  /admin/login                  ─── public
  /admin/*                      ─── <AdminLayout> 가드 + 8 + 19 sub
  *                             ─── <NotFoundPage>
```

## 2. router 정의 (`src/router.tsx`)

```tsx
export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <NotFoundPage />,
    children: [{ index: true, element: <HomePage /> }],
  },

  // 학생 부트스트랩 (가드 없음)
  { path: '/high-student/login',   element: <HighStudentLogin /> },
  { path: '/high-student/signup',  element: <HighStudentSignup /> },
  { path: '/high-student/connect', element: <HighStudentConnect /> },
  { path: '/high-student/pending', element: <HighStudentPending /> },

  // 학생 본체 (Layout 이 가드)
  {
    path: '/high-student',
    element: <HighStudentLayout />,
    children: [
      { index: true,            element: <HighStudentRoadmap /> },
      { path: 'roadmap',        element: <HighStudentRoadmap /> },
      { path: 'topic',          element: <HighStudentTopicList /> },
      // ... 13 sub
    ],
  },

  // /middle-student/... 동일 패턴
  // /master/... 동일
  // /admin/... 동일
])
```

`App.tsx` → `<AppRouter />` → `<RouterProvider router={router} />`.

## 3. 인증 가드 패턴

### 3.1 master / admin — atom 기반 단순 가드

```tsx
export function MasterLayout() {
  const isAuthed = useAtomValue(isMasterAuthenticatedAtom)

  if (!isAuthed) {
    return <Navigate to="/master/login" replace />
  }

  return (
    <div className="flex h-screen bg-[#F8FAFC]">
      <MasterSidebar />
      <main className="flex-1 overflow-y-auto"><Outlet /></main>
    </div>
  )
}
```

`isMasterAuthenticatedAtom`:

```ts
export const isMasterAuthenticatedAtom = atom((get) => {
  const token = get(masterTokenState)
  if (!token.accessToken) return false
  if (!token.expiresIn) return false
  return new Date(token.expiresIn).getTime() > Date.now()
})
```

- token + 만료 시점 검증.
- `atomWithStorage('masterToken', ...)` 로 localStorage 동기화 → 새로고침 후에도 세션 유지.
- 만료 시 자동 `<Navigate>` (token expired 검증).

### 3.2 admin — 동일 패턴

`isAdminAuthenticatedAtom` 가 `adminTokenAtom` 검증. role 은 `AdminRole = 'OWNER' | 'TEACHER'`
(teacher 페이지는 후속).

### 3.3 학생 (high / middle) — 4 status 분기

학생 측은 단순 가드가 아니라 status 별 redirect:

```ts
// _shared/auth/useStudentLogin.ts
//
//   PENDING_CONNECT → /<group>/connect (학원 코드 입력)
//   PENDING_APPROVE → /<group>/pending (학원장 승인 대기)
//   ACTIVE          → /<group>/roadmap
//   REJECTED        → 에러 메시지 (login 화면 머무름)
```

이유: 학생 가입 흐름이 multi-step.

1. `signup` → account 생성 (status=PENDING_CONNECT)
2. `connect` → 학원 코드 입력 → membership row (status=PENDING_APPROVE)
3. 학원장 `student-approval` → membership ACTIVE → account status=ACTIVE
4. 그 후 정상 학습 페이지 진입

login 시 status 검사하여 적절한 path 로 navigate. `StudentLayout` 도 status=ACTIVE 가 아니면
바이패스 (status 검사 atom 도 있음).

## 4. <Navigate replace />

`replace` 사용 — history 에 login 이 push 되지 않도록. 뒤로가기 시 무한 redirect 회피.

```tsx
<Navigate to="/master/login" replace />
```

## 5. 외부 진입 (URL 직접 입력 / 북마크)

- `/master/billing` 직접 진입 → `MasterLayout` 이 가드 → `/master/login` 으로 redirect.
- login 후 원래 의도 path 로 복귀? **현재 미구현** — login 후 항상 `/master` (index) 로 이동. 후속 작업 후보 (location state 로 redirect target 전달).

## 6. 404 처리

`errorElement: <NotFoundPage />` — `RootLayout` 의 errorElement. children 어디서 에러나도 fallback.
react-router 7 의 errorElement 위치는 root 한 곳에만.

## 7. 신규 라우트 추가 체크리스트

1. role 결정 — 어느 sub-tree?
2. `pages/{role}/_pages/{feature}/{ComponentName}.tsx` 작성.
3. `src/router.tsx`:
   - public 부트스트랩이면 children 밖에 `{ path: '/role/feature', element: <Comp /> }` 한 줄.
   - 가드 안이면 `pages/{role}/_layout/{Role}Layout.tsx` 의 `<Outlet>` children 배열에 추가.
4. 사이드바 메뉴 (`{Role}Sidebar.tsx`) — 일반적으로 같은 PR 또는 별도 PR.
5. 직접 진입 시 가드 동작 확인.

## 8. lazy / code-split (현재 없음)

- 모든 라우트가 eager import (`src/router.tsx` 상단의 import 60+ 줄).
- 번들 사이즈: 1.4MB / gz 380KB — admin + master + 양 role student 단일 chunk.
- BE 연동 시점에 `React.lazy(() => import('...'))` + `<Suspense>` 도입 검토 ([[bkurs-fe#5]]).

## 9. router import 60+ 줄

현재 `router.tsx` 의 상단이 60+ 줄의 named import. **lazy 도입 시 해소** — 그 전까지는 그대로
두는 게 더 안전 (lazy 가 의도와 다르게 잘리지 않도록).

## 10. 디버깅 — "왜 자꾸 login 으로 튕기지"

체크 순서:

1. localStorage `masterToken` / `adminToken` / `studentToken` 값 — 존재? 만료?
2. `isMasterAuthenticatedAtom` derived 가 false 인 이유 — `accessToken` 누락 / `expiresIn` 과거?
3. login 후 navigate 가 제대로 atom 갱신? Jotai DevTools 또는 console.log atom getter.
4. 401 응답 → `apiClient` interceptor 가 `setAccessToken(null)` 호출 → 다음 요청 시 401 반복 → 다음 페이지 진입 시 redirect.

## cross-link

- [[bkurs-fe]]
- [[directory-structure]] — router / app / pages 위치
- [[role-modules]] — 4 role 페이지 책임
- [[state-and-auth]] — atom + status 흐름 상세
- [[lib-layer-guide]] — 401 처리 + token storage
- [[../bkurs-be/auth-and-security]] — BE 측 인증 흐름
