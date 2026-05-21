---
title: "bkurs-fe — 디렉토리 구조 인수인계"
kind: handover
status: stable
created_at: 2026-05-21T22:45:00+09:00
tags: [bkurs-fe, handover, structure, frontend]
related:
  - "[[bkurs-fe]]"
  - "[[role-modules]]"
  - "[[lib-layer-guide]]"
  - "[[shared-components]]"
  - "[[routing-and-protection]]"
home_hub: bkurs-fe
---

# bkurs-fe — 디렉토리 구조 인수인계

> 신규 합류자 1순위 가이드. "어디에 뭐가 있나" + "role 별 페이지는 어디 / 공통은 어디 /
> API client 와 hook 컨벤션". 깊은 내용은 각 sub-doc.

## 1. 레포 최상단

```
bkurs-fe/
├── .env.example                # VITE_API_BASE_URL / VITE_SUPABASE_URL 등 — 실 값은 .env.local (gitignore)
├── .github/workflows/          # CI (Node 20 + npm ci + typecheck + build + lint)
├── .vscode/                    # 권장 설정 (ESLint format on save)
├── .nvmrc                      # Node 20
├── .gitmessage.txt             # commit 템플릿 (한국어 + gitmoji + 3 섹션)
├── .prettierrc / .prettierignore
├── eslint.config.js            # flat config (TS strict + React 17 jsx-runtime)
├── index.html                  # Vite entry
├── vite.config.ts              # alias @ / build target / dev server
├── tsconfig*.json              # base / app / node 분리
├── tailwind.config.js          # brand 토큰 (high=blue, middle=green, master=purple)
├── postcss.config.js
├── package.json / package-lock.json
├── public/                     # 정적 리소스 (favicon 등)
├── docs/
│   ├── BRANCH_STRATEGY.md
│   ├── COMMIT_CONVENTION.md
│   ├── PROJECT_STRUCTURE.md    # 본 문서의 코드측 SSoT 와 일치 유지
│   ├── RELEASE_FLOW.md
│   └── CHANGELOG.md
└── src/                        # 본 문서 §2~§5
```

스크립트 (package.json):

| 명령 | 설명 |
| --- | --- |
| `npm run dev` | Vite dev (HMR, http://localhost:5173) |
| `npm run build` | `tsc -b --noEmit --force && vite build` |
| `npm run preview` | build 결과 로컬 미리보기 |
| `npm run typecheck` | `tsc -b --noEmit --force` ([[build-and-ci#tsbuildinfo]]) |
| `npm run lint` / `lint:fix` | ESLint |
| `npm run format` | Prettier |

## 2. `src/` 최상단

```
src/
├── main.tsx                # 진입 — createRoot + <StrictMode><App /></StrictMode>
├── App.tsx                 # <AppProviders><AppRouter /></AppProviders>
├── app/
│   ├── AppProviders.tsx    # QueryClient / Jotai / AntD ConfigProvider (koKR + antdTheme)
│   └── AppRouter.tsx       # RouterProvider(router) — router 자체는 src/router.tsx
├── router.tsx              # createBrowserRouter — role-based 4 sub-tree + login bootstrap
├── pages/                  # role 별 page 컴포넌트 (§3)
├── lib/                    # 공용 logic (§4)
├── components/             # 공용 컴포넌트
│   └── layout/RootLayout.tsx
├── styles/
│   ├── index.css           # Tailwind base + 글로벌 한 줄
│   └── antd-theme.ts       # AntD ThemeConfig (token + components)
├── api/                    # (legacy 디렉토리, 현재 비어있음 — lib/api 가 SSoT)
├── hooks/                  # (legacy, 비어있음)
├── store/                  # (legacy, 비어있음)
├── types/                  # (legacy, 비어있음)
└── constants/              # (필요 시)
```

> ⚠️ `api/`, `hooks/`, `store/`, `types/` 는 inroad-high 초기 분기 때 생성된 placeholder.
> 현재 모든 logic 은 `lib/` 하위로 통합됨. 새 코드는 **lib/** 에 둔다 — 본 4 디렉토리는 결국
> 삭제 후보.

## 3. `src/pages/` — role 별 sub-tree

```
pages/
├── common/                 # role 무관 (HomePage, NotFoundPage)
├── _shared/                # role 간 공용 hook (auth 4)
│   └── auth/
│       ├── useStudentLogin.ts
│       ├── useStudentSignup.ts
│       ├── useStudentConnect.ts
│       └── useStudentPending.ts
├── admin/                  # 학원장 (OWNER role, mock 단계는 supabase 의존)
│   ├── _layout/
│   ├── _pages/
│   │   ├── academy/        # 학원 정보
│   │   ├── announcements/  # 공지 관리
│   │   ├── billing/
│   │   ├── dashboard/
│   │   ├── login/
│   │   ├── settings/
│   │   ├── student-approval/
│   │   ├── students/       # 학생 목록 + 상세 (high-tabs / middle-tabs)
│   │   │   └── detail/
│   │   │       ├── high-tabs/{expect,past}/        # 고등 학생 상세 탭
│   │   │       └── middle-tabs/{middle-*}/         # 중등 학생 상세 탭
│   │   ├── suhaeng/
│   │   └── teachers/
│   └── _store/auth.ts
├── master/                 # 본사 (MASTER role)
│   ├── _layout/
│   ├── _pages/{academies,audit,billing,dashboard,data,lessons,login,notices,questions,staff}
│   └── _store/auth.ts
├── high-student/           # 고등 학생 (STUDENT_HIGH)
│   ├── _layout/
│   ├── _components/ThreadView.tsx
│   ├── _data/              # mock data (jotai supabase 전환 전 임시)
│   └── _pages/{book,concept,connect,expect,login,major,mockexam,past,pending,presentation,record,roadmap,signup,simulation,suhaeng,topic}
└── middle-student/         # 중등 학생 (STUDENT_MIDDLE)
    ├── _layout/
    ├── _components/ThreadView.tsx
    ├── _data/
    └── _pages/{book,concept,connect,debate,expect,homework,lesson,login,past,pending,presentation,record,roadmap,signup,simulation,suhaeng}
```

규칙 (자세히 [[role-modules]]):

- `_layout/` — `<Outlet>` + 사이드바 + 인증 가드 (`<Navigate to="/{role}/login" />`)
- `_pages/{feature}/` — 한 라우트 = 한 폴더. `index` 컴포넌트 (또는 동명 컴포넌트) + sub-component
- `_components/` — **본 role 내** 재사용 컴포넌트 (예: `ThreadView`)
- `_data/` — mock data (supabase / mock 단계, 본 BE 도입 후 교체)
- `_store/` — role 한정 jotai atom (admin/master `auth.ts`)

cross-role 재사용은 `pages/_shared/` 또는 `lib/` 로 승격.

## 4. `src/lib/` — 공용 logic

```
lib/
├── env.ts                  # VITE_* 환경 변수 normalize
├── http.ts                 # legacy fetch wrapper (점진 제거)
├── supabase.ts             # supabase client (legacy, 점진 제거 — TODO bkurs-fe.md §4)
├── api/                    # bkurs-be REST client 25 개
│   ├── client.ts           # axios 인스턴스 (Bearer + 401 처리)
│   ├── envelope.ts         # ApiEnvelope / ApiError / ApiClientError + unwrap()
│   ├── index.ts            # 전 client re-export
│   └── {domain}.ts × 22    # 도메인별 client
├── hooks/                  # react-query wrapper 18 개
│   ├── index.ts            # 재export
│   └── use{Domain}.ts × 18
├── auth/
│   ├── studentAuth.ts      # studentProfileAtom + studentTokenAtom + 4 status enum
│   └── atoms.ts            # legacy shim (점진 제거)
└── utils/
    └── grade.ts            # 학년 라벨 등
```

자세히 → [[lib-layer-guide]].

## 5. provider 조립 (`app/AppProviders.tsx`)

```tsx
<QueryClientProvider client={queryClient}>     // TanStack Query
  <JotaiProvider>                              // Jotai (atom 격리)
    <ConfigProvider theme={antdTheme} locale={koKR}>  // AntD 6 (token + 한국어)
      {children}
    </ConfigProvider>
  </JotaiProvider>
  {DEV && <ReactQueryDevtools />}
</QueryClientProvider>
```

QueryClient 기본값:
- `staleTime: 60s` — 일반 조회 1분 캐시
- `refetchOnWindowFocus: false` — 탭 전환 시 자동 refetch 끔
- `retry: 1` — 실패 시 한 번만 재시도

## 6. 빠른 탐색 cheat sheet

| 찾고 싶은 것 | 어디로 |
| --- | --- |
| "이 URL 어디서?" | `src/router.tsx` 의 path → page import 확인 |
| "이 페이지 컴포넌트?" | `src/pages/{role}/_pages/{feature}/` |
| "이 API 호출 어디서 정의?" | `src/lib/api/{domain}.ts` |
| "이 데이터 fetch 어디서?" | `src/lib/hooks/use{Domain}.ts` (react-query) |
| "왜 toast 가 안 뜨지?" | `AppProviders.tsx` + AntD `message` API 사용 여부 |
| "role 별 인증 가드?" | `pages/{role}/_layout/{Role}Layout.tsx` + `_store/auth.ts` |
| "design token (색 / 간격 / 폰트)?" | `tailwind.config.js` 의 brand-* + `styles/antd-theme.ts` |
| "API base URL?" | `lib/api/client.ts` (`VITE_API_BASE_URL`) |

## cross-link

- [[bkurs-fe]] — 프로젝트 hub
- [[role-modules]] — admin / master / high-student / middle-student / _shared 책임
- [[lib-layer-guide]] — lib/api + lib/hooks 컨벤션
- [[shared-components]] — ThreadView / AiAnalysisPanel / ScoreRadar 재사용
- [[routing-and-protection]] — router + 인증 가드
- [[design-system]] — Tailwind + AntD 토큰
- [[state-and-auth]] — Jotai + supabase 점진 제거 + 4 role 흐름
- [[build-and-ci]] — vite / tsc / ESLint / GitHub Actions
