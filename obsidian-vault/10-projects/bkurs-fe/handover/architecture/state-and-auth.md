---
title: "bkurs-fe — 상태 관리 + 인증 흐름"
kind: handover
status: stable
created_at: 2026-05-21T23:15:00+09:00
tags: [bkurs-fe, handover, jotai, react-query, auth, supabase]
related:
  - "[[bkurs-fe]]"
  - "[[lib-layer-guide]]"
  - "[[routing-and-protection]]"
  - "[[role-modules]]"
home_hub: bkurs-fe
---

# bkurs-fe — 상태 관리 + 인증 흐름

> Jotai (atom) + TanStack Query (서버 캐시) 의 책임 분리 + 4 role 인증 atom 위치 + supabase
> 점진 제거 진행 상황. "왜 학생 로그인 후 4가지 페이지로 분기되는가" 도 본 문서에서.

## 1. 한 장 요약

| 책임 | 도구 | 위치 |
| --- | --- | --- |
| 인증 token / 프로필 (long-lived) | Jotai `atomWithStorage` | `lib/auth/`, `pages/{role}/_store/auth.ts` |
| 서버 데이터 캐시 (refetch / invalidate) | TanStack Query (`useQuery` / `useMutation`) | `lib/hooks/` |
| UI 일시 상태 (form input, modal open) | `useState` / `useRef` (React) | 컴포넌트 내부 |

> **Jotai 와 React Query 동시 사용 이유**: 서버 캐시 (Query) 와 클라이언트 영속 상태 (atom) 는
> 책임이 다르다. Jotai 의 `atomWithStorage` 로 localStorage 동기화 — 새로고침 후에도 token 유지.

## 2. atom 위치 4 종

### 2.1 master — `pages/master/_store/auth.ts`

```ts
export interface MasterToken { accessToken?: string; expiresIn?: string }
export type MasterRole = 'SUPER_ADMIN' | 'STAFF'
export interface Master { id?, name?, email?, role: MasterRole }

export const masterTokenState   = atomWithStorage<MasterToken>('masterToken',  EMPTY_TOKEN)
export const masterState        = atomWithStorage<Master>('masterInfo', DEFAULT_MASTER)
export const isMasterAuthenticatedAtom = atom((get) => {
  const t = get(masterTokenState)
  return !!t.accessToken && !!t.expiresIn && new Date(t.expiresIn).getTime() > Date.now()
})
```

### 2.2 admin — `pages/admin/_store/auth.ts`

```ts
export type AdminRole = 'OWNER' | 'TEACHER'
export type AcademyPlan = 'high' | 'middle'

export interface AdminToken { accessToken?: string; expiresIn?: string }
export interface Academy { academyId?, academyCode?, academyName?, ownerName?, role, teacherId?, plans }

export const adminTokenAtom    = atomWithStorage<AdminToken>(...)
export const academyAtom       = atomWithStorage<Academy>(...)
export const isAdminAuthenticatedAtom = atom((get) => ...)
```

### 2.3 학생 (high + middle 공용) — `lib/auth/studentAuth.ts`

```ts
export type StudentRole = 'high_student' | 'middle_student'
export type StudentStatus = 'pending' | 'active' | 'rejected'

export interface StudentToken { accessToken?, expiresIn? }
export interface StudentProfile {
  id?, email?, name?, role?, grade?, schoolName?,
  academyCode?, academyName?, status: StudentStatus, parentPhone?
}

export const studentTokenAtom   = atomWithStorage<StudentToken>('studentToken', ...)
export const studentProfileAtom = atomWithStorage<StudentProfile>('studentProfile', ...)
```

high / middle 공용 — `role` 필드로 구분.

### 2.4 legacy shim — `lib/auth/atoms.ts`

```ts
export const studentState   = studentProfileAtom    // alias
export const academyState   = atom((get) => { ... })// derived from studentProfileAtom
```

> inroad-high 의 `lib/auth/atoms` 경로를 import 하는 mock 단계 코드 (concept 등) 와 호환. 신규
> 코드는 `studentAuth.ts` 직접.

## 3. atomWithStorage — localStorage 동기화

```ts
atomWithStorage<MasterToken>('masterToken', EMPTY_TOKEN)
//                            ↑ localStorage key
```

- 새로고침 후 atom 자동 복원
- 다른 탭에서 변경 시 storage event 로 sync (Jotai 가 처리)
- key 충돌 금지 — `masterToken / adminToken / studentToken / studentProfile / masterInfo / academy*`

## 4. 학생 4 hook — `pages/_shared/auth/`

### 4.1 `useStudentLogin(group: 'high' | 'middle')`

흐름:

```
1. authApi.login({ email, password }) 호출
2. role 검증 — 응답 role 이 STUDENT_HIGH (group='high') / STUDENT_MIDDLE (group='middle') 인지
   - 불일치 → "고등/중등 계정만 가능" 에러
3. status 별 navigate:
   - PENDING_CONNECT → /<group>/connect (학원 코드 입력 화면)
   - PENDING_APPROVE → /<group>/pending (학원장 승인 대기 화면)
   - ACTIVE          → /<group>/roadmap
   - REJECTED        → 에러 메시지, login 화면 유지
4. studentProfileAtom + studentTokenAtom 갱신 (atomWithStorage 가 자동 localStorage 저장)
```

핵심: **status 가 단순 boolean 가드가 아니라 4 갈래 분기**.

### 4.2 `useStudentSignup(group)`

`authApi.signup({ email, password, role, ... })` + role 매핑 (`high → STUDENT_HIGH`).
응답 status=PENDING_CONNECT 받아서 `/<group>/connect` 로 이동.

### 4.3 `useStudentConnect(group)`

`/api/v1/auth/connect` 호출 — 학원 코드 입력. 성공 시 academy_code → academyApi.connect 또는
authApi.connect. status=PENDING_APPROVE 받아서 `/<group>/pending` 으로.

### 4.4 `useStudentPending(group)`

학원장 승인 대기 화면의 hook. polling 또는 manual refresh 로 `studentProfileAtom.status` 검사.
ACTIVE 되면 roadmap.

## 5. 4 status 도메인 매핑

| FE StudentStatus | BE AccountStatus + MembershipStatus | UI 화면 |
| --- | --- | --- |
| `PENDING_CONNECT` | account ACTIVE + membership 없음 | `/connect` |
| `PENDING_APPROVE` | account ACTIVE + membership PENDING | `/pending` |
| `ACTIVE` | account ACTIVE + membership ACTIVE | `/roadmap` (정상) |
| `REJECTED` | membership REJECTED 또는 account BLOCKED | login 에러 |

> BE [[../bkurs-be/domain-catalog#auth]] 의 Role / Status enum 과 1:1.

## 6. supabase 점진 제거

inroad-high 가 BaaS 로 supabase 직접 호출했던 잔재가 일부 남아있다 (mock 단계 보조):

| 위치 | 상태 |
| --- | --- |
| `lib/supabase.ts` | 클라이언트만 생성. 호출은 후속 제거 대상 |
| `lib/auth/atoms.ts` | legacy shim — concept 가 hook 으로 swap 되면 제거 |
| concept / book / 일부 mock 화면 | 점진 swap 진행 — [[screen-logic-and-api-map]] |
| `.env.local` 의 `VITE_SUPABASE_URL / VITE_SUPABASE_ANON_KEY` | 미설정이어도 동작 — warn 만 |

**제거 절차**:

1. 해당 mock 화면이 사용하는 데이터를 BE endpoint 로 매핑.
2. `lib/api/{domain}.ts` 와 `lib/hooks/use{Domain}.ts` 가 있는지 확인.
3. 화면의 supabase 호출 → hook 호출로 교체.
4. mock 데이터 (`_data/...mock.ts`) 의존 제거.
5. supabase import 0 이면 `lib/supabase.ts` 제거 + `.env.example` 에서 키 삭제.

진행 추적: [[bkurs-fe#5]] 의 "Supabase wrapper 제거" 항목.

## 7. React Query 캐시 정책

`AppProviders.tsx`:

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,           // 60s — 1분 캐시 hit
      refetchOnWindowFocus: false,    // 탭 전환 시 refetch 끔
      retry: 1,                       // 실패 시 1회 재시도
    },
  },
})
```

- mutation 후 `invalidateQueries({ queryKey: ['homework', 'mine'] })` 호출 → 다음 mount 시 refetch.
- optimistic update: `setQueryData<T[]>(key, prev => updated)` ([[lib-layer-guide#4.3]]).
- 페이지 unmount 시 cache 유지 — 다음 진입 시 즉시 표시 + 백그라운드 refetch.

## 8. 401 흐름

1. axios interceptor: `error.response?.status === 401` → `setAccessToken(null)` (localStorage clear).
2. 다음 페이지 진입 시 `is{Role}AuthenticatedAtom` false → Layout 의 `<Navigate to="/{role}/login" />`.
3. login 후 atom 갱신 → 정상 흐름.

> refresh 흐름은 **후속 cycle** ([[../bkurs-be/auth-and-security#4]] 의 BE 측 rotation 은 존재하지만
> FE 측 자동 refresh interceptor 는 미구현).

## 9. 신규 atom 추가 시 결정 트리

```
이 상태가 어디서 쓰이나?
├── 한 컴포넌트 내부만 → useState
├── 같은 page 안 (props drilling 회피) → useReducer 또는 페이지 한정 atom (페이지 폴더 내)
├── 같은 role 의 여러 페이지 → pages/{role}/_store/{name}.ts
├── 여러 role → lib/auth/ 또는 lib/{name}/ 로 승격
└── 서버 데이터 ↔ 캐시 ↔ refetch 가 필요 → atom 이 아니라 react-query useQuery
```

## 10. 페이지에서 atom + hook 동시 사용 예

```tsx
function HighStudentRoadmap() {
  const profile = useAtomValue(studentProfileAtom)     // 학생 본인 정보 (localStorage)
  const { data: courses, isLoading } = useMyHomework() // 서버 데이터 (react-query)

  if (!profile.id) return <Navigate to="/high-student/login" replace />
  if (isLoading) return <Spin />
  return <RoadmapList items={courses} studentName={profile.name} />
}
```

## cross-link

- [[bkurs-fe]]
- [[lib-layer-guide]] — react-query / axios 컨벤션
- [[routing-and-protection]] — atom 가드 + 4 status 분기
- [[role-modules]] — role 별 _store 위치
- [[screen-logic-and-api-map]] — supabase → hook swap 진행
- [[../bkurs-be/auth-and-security]] — BE 인증 흐름 (status / refresh)
