---
title: "Role-Based Layout — role 별 RootLayout / Sidebar / 보호 라우트 패턴"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, routing, layout, auth, recipes]
related:
  - "[[react-router-v6]]"
  - "[[jotai]]"
  - "[[auth]]"
created_at: 2026-05-18
---

# Role-Based Layout — role 별 RootLayout / Sidebar / 보호 라우트 패턴

**[[recipes|↑ Recipes hub]]**

> 한 프로덕트에 admin / staff / 학생 / 운영자 같은 **여러 role** 이 있을 때, 각 role 마다 Header/Sidebar 가 다르고 권한 경계도 다르다. 이걸 **Layout 컴포넌트 + 보호 라우트 + role 별 auth atom** 으로 깔끔히 나누는 패턴.

## 1. 왜 role 별 Layout 이 필요한가

### 왜 필요
- role 별 UI 가 다르다 (메뉴 / 색상 / 네비게이션 깊이). 한 Layout 에 if/else 로 분기하면 100줄 안에 4개 role 의 사이드바가 섞임 → 변경 비용 폭증.
- role 별 권한 가드가 다르다 (master 는 `SUPER_ADMIN` 또는 `STAFF` 만, admin 은 학원 owner 만). 한 `ProtectedRoute` 에 role 옵션을 받아 if 분기 → 라우터 트리에서 어떤 role 이 어느 페이지 보는지 한눈에 안 보임.

### 안 하면 문제
- 어드민 사이드바를 손대다 마스터 사이드바가 회귀.
- 학생 페이지인데 어드민이 접근 가능 — 권한 경계 사고.
- 신규 role 추가 시 한 Layout 의 if/else 가 5중첩 → 리뷰 불가.

### 대안
- **단일 Layout + role-config**: 한 Layout 이 `roleConfig[role].menu / theme / guard` 를 가져다 렌더. 작은 차이만 있을 때 적합. 차이가 크면 (메뉴 구조 자체가 다름) 결국 if/else 폭발.
- **role 별 Layout + Outlet** (본 패턴): 각 role 마다 `<RoleLayout>` 컴포넌트를 별도로 두고 router 트리에서 분기. 차이가 클 때 우월.
- **별도 SPA** (role 별 빌드): 너무 분리. 공통 컴포넌트 / 빌드 / 배포 비용 증가.

### 트레이드오프
- 본 패턴: 공통 컴포넌트 (atom Button 등) 는 `src/components/atom/` 에 두고, role 별 Sidebar/Layout 은 `src/pages/<role>/_layout/` 에 둠. 한 role 변경이 다른 role 에 누수되지 않음.

## 2. 디렉토리 구조

```
src/pages/<role>/
├── _layout/
│   ├── <Role>Layout.tsx       # 인증 가드 + Outlet host (얇음)
│   ├── <Role>Sidebar.tsx      # 사이드바 UI
│   ├── theme.ts               # role 별 디자인 상수 (인라인 style 용)
│   └── menu.ts                # 메뉴 항목 정의
├── _store/
│   └── auth.ts                # Jotai atomWithStorage — role 별 세션
└── _pages/
    ├── login/                 # role 별 Login (가드 밖)
    └── <feature>/             # 가드 안에서 보호되는 페이지들
```

`_` prefix 는 "라우트 매핑이 아닌 내부 자원" 을 표시 (route component 와 시각적 구분).

## 3. role 별 auth atom

```ts
// src/pages/master/_store/auth.ts
import { atomWithStorage } from 'jotai/utils'
import { atom } from 'jotai'

export interface MasterToken {
  accessToken: string | undefined
  expiresIn: string | undefined
}

export const masterTokenState = atomWithStorage<MasterToken>('masterToken', {
  accessToken: undefined,
  expiresIn: undefined,
})

export const isMasterAuthenticatedAtom = atom((get) => {
  const token = get(masterTokenState)
  if (!token.accessToken || !token.expiresIn) return false
  return new Date(token.expiresIn).getTime() > Date.now()
})
```

핵심:
- `atomWithStorage` 로 localStorage 자동 동기화 (새로고침 / 탭 재오픈 후 세션 유지).
- key prefix 를 role 별로 분리 (`masterToken`, `adminToken`, `studentToken`) — 다른 role 의 세션이 섞이지 않음.
- 파생 atom `isXAuthenticatedAtom` 으로 만료 여부 계산 — 컴포넌트는 boolean 만 본다.

## 4. Layout 컴포넌트 — 인증 가드 + Outlet host

```tsx
// src/pages/master/_layout/MasterLayout.tsx
import { Outlet, Navigate } from 'react-router-dom'
import { useAtomValue } from 'jotai'
import { isMasterAuthenticatedAtom } from '@/pages/master/_store/auth'
import { MasterSidebar } from './MasterSidebar'

export function MasterLayout() {
  const isAuthed = useAtomValue(isMasterAuthenticatedAtom)
  if (!isAuthed) return <Navigate to="/master/login" replace />

  return (
    <div className="flex h-screen bg-[#F8FAFC]">
      <MasterSidebar />
      <main className="flex-1 overflow-y-auto">
        <Outlet />
      </main>
    </div>
  )
}
```

- **얇게 유지** — 인증 가드 + Sidebar + Outlet 3개만. 추가 책임 (헤더 알림 / 검색 / 브레드크럼) 은 별도 컴포넌트로.
- `<Navigate replace>` 로 history 누적 방지.

## 5. 라우터 매핑

```tsx
// src/router.tsx
export const router = createBrowserRouter([
  // 공용 (랜딩 / 404)
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <NotFoundPage />,
    children: [{ index: true, element: <HomePage /> }],
  },
  // role 별 login (가드 밖 — 미인증 진입 가능)
  { path: '/master/login', element: <MasterLogin /> },
  // role 별 보호 영역 (Layout 안 자동 가드)
  {
    path: '/master',
    element: <MasterLayout />,
    children: [
      { index: true, element: <MasterDashboard /> },
      { path: 'academies', element: <Academies /> },
      // ...
    ],
  },
])
```

- login 라우트는 Layout 밖 — 미인증 사용자가 진입해야 하므로.
- 보호 영역 (`/master`) 은 Layout 안에 children 으로 → Layout 의 가드 한 번이 전체 영역 보호.

## 6. role 별 theme / menu 상수 분리

한 Sidebar 안에 색 / 메뉴 / 로고를 다 박지 말고, **상수만 별도 모듈** 에:

```ts
// src/pages/master/_layout/theme.ts
export const MASTER_THEME = {
  accent: '#7C3AED',
  gradient: 'linear-gradient(135deg, #5B21B6, #8B5CF6)',
  // ...
} as const

// src/pages/master/_layout/menu.ts
export const MASTER_MENU = [
  { path: '/master', label: '대시보드' },
  { path: '/master/academies', label: '학원 관리' },
  // ...
]
```

- 색상은 가능하면 Tailwind config 의 `brand-<role>` 토큰 (`brand-master`, `brand-high`, `brand-middle`) 으로 끌어올림. 인라인 헥사가 필요한 경우 (CSS gradient / radial / dynamic boxShadow) 만 theme.ts 에.
- 메뉴 추가/라벨 변경은 menu.ts 한 곳만 수정 → Sidebar 자동 반영.

## 7. 4 role 일 때 폴더 구조

```
src/pages/
├── admin/          # 학원 어드민
│   ├── _layout/    # AdminLayout / AdminSidebar
│   ├── _store/     # adminTokenState
│   └── _pages/
├── high-student/   # 고등 학생
│   ├── _layout/
│   ├── _store/
│   └── _pages/
├── middle-student/ # 중등 학생 (high 와 비슷하지만 색 다름)
└── master/         # 본사 마스터
```

공통 컴포넌트 (`Button`, `Modal`, `Card`) 는 `src/components/` 에 — role 무관.

## 8. 시니어 분리 신호 (Layout 이 비대해질 때)

다음 중 하나라도 발견되면 분리:
- Layout 한 파일이 **100줄 초과** + 사이드바 / 헤더 / 가드 / 알림 / 검색 등 4가지 책임 이상
- Sidebar 안에 메뉴 / 유저 패널 / 로그아웃 / 알림 등 3가지 이상
- 색상 / 메뉴 / 라벨이 컴포넌트 안에 인라인 → 상수 모듈로

## 9. 함정

1. **모든 role 을 한 Layout 에 if** — 100줄 안에 4 role 사이드바 섞임. 무조건 role 별 Layout.
2. **login 도 Layout 안에 두기** — 미인증이 가드에 막혀 무한 redirect.
3. **auth atom key 공유** — 학원 어드민 토큰이 마스터 세션에 흘러들어 권한 사고.
4. **`useEffect` 안에서 navigate** — Layout 의 가드는 render 시점에 `<Navigate>` 로 처리. useEffect 안에 navigate 두면 첫 frame 깜빡임.
5. **menu / theme 상수를 Sidebar 안에 인라인** — 다른 컴포넌트 (헤더 / 모바일 메뉴) 에서 같은 메뉴 쓸 때 중복.

## 10. 관련

- [[react-router-v6]] — `createBrowserRouter` / `Navigate` / `Outlet` 깊이
- [[jotai]] — `atomWithStorage` 의 localStorage 동기화 동작
- [[auth]] — 인증 흐름 일반
- [[project-structure]] — `_` prefix 디렉토리 컨벤션
