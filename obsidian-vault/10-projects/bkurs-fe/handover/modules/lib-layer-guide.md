---
title: "bkurs-fe — lib/ 레이어 인수인계"
kind: handover
status: stable
created_at: 2026-05-21T22:55:00+09:00
tags: [bkurs-fe, handover, api-client, react-query, axios]
related:
  - "[[bkurs-fe]]"
  - "[[directory-structure]]"
  - "[[architecture-api-foundation]]"
  - "[[decisions/decision-hook-convention]]"
home_hub: bkurs-fe
---

# bkurs-fe — lib/ 레이어 인수인계

> `src/lib/` 의 4 sub-layer (api / hooks / auth / utils) 와 컨벤션. 새 BE 도메인 들어올 때 어디
> 추가 / 어떤 패턴 따를지. [[architecture-api-foundation]] 가 도입 결정의 ADR, 본 문서는 **현재
> 컨벤션 SSoT**.

## 0. 한 장 요약

```
도메인 BE endpoint
    ▼
lib/api/{domain}.ts  ─── axios 호출 + ApiEnvelope unwrap
    │  (예: homeworkApi.listMine())
    ▼
lib/hooks/use{Domain}.ts ─── react-query wrapper
    │  (예: useMyHomework() / useSubmitHomework())
    ▼
페이지 컴포넌트 (pages/{role}/_pages/...)
```

규칙:
- 페이지는 **hook 만** 호출. axios 직접 호출 금지.
- hook 은 axios client 만 호출. `fetch` / 다른 라이브러리 금지.

## 1. `lib/api/client.ts` — axios 인스턴스

| 항목 | 값 |
| --- | --- |
| `baseURL` | `import.meta.env.VITE_API_BASE_URL ?? '/api'` |
| `timeout` | 15s |
| `Content-Type / Accept` | `application/json` |

### 1.1 인증 — Bearer 자동 부착

`apiClient.interceptors.request.use(attachAuth)` 가 매 요청에 `Authorization: Bearer <token>` 부착.
token 은 `localStorage('bkurs.accessToken')`.

```ts
export function getAccessToken(): string | null { ... }
export function setAccessToken(token: string | null): void { ... }
```

### 1.2 401 처리

`apiClient.interceptors.response.use` — `error.response?.status === 401` 이면 `setAccessToken(null)`
호출 (localStorage clear). refresh 흐름은 **후속 cycle** — 현재는 401 이면 atom 리셋 + login 이동
(페이지 측).

### 1.3 에러 envelope → `ApiClientError` throw

응답 envelope 에 `error` 가 있으면 client interceptor 가 `ApiClientError(error)` 로 reject.
페이지에서:

```ts
import type { ApiClientError } from '@/lib/api/envelope'

try {
  await homeworkApi.submit(id, content)
} catch (e) {
  const ce = e as ApiClientError
  message.error(ce.message)
  if (ce.code === 'HMW-4001') { /* 도메인별 처리 */ }
}
```

## 2. `lib/api/envelope.ts` — 응답 타입

```ts
interface ApiEnvelope<T> { success: boolean; data?: T; error?: ApiError; meta: ApiMeta }
interface ApiError      { code: string; message: string; details?: Record<string,unknown>; field?: string }
interface ApiMeta       { requestId: string; timestamp: string; durationMs?: number }

function unwrap<T>(envelope: ApiEnvelope<T>): T { /* throw ApiClientError on !success */ }
class ApiClientError extends Error { code; details; field }
```

BE 의 `ApiEnvelope<T>` 와 1:1. 차이:
- BE 의 `error.fieldErrors` (배열) vs FE 의 `error.field` (단일) — BE 가 fieldErrors[0] 을 단일 field 로 매핑하는 경우 / FE 는 details 안에 list 가능. 통일은 후속 envelope migration 진행 따라.

## 3. `lib/api/{domain}.ts` — 25 도메인 client

현재 client (PR # 누적):

```
academy.ts, announcement.ts, auth.ts, concept.ts, dashboard.ts, discussion.ts,
homework.ts, interview.ts, majorQuestion.ts, pastQuestion.ts, performance.ts,
plan.ts, question.ts, reading.ts, record.ts, saenggibu.ts, search.ts,
simulation.ts, statement.ts, stats.ts, studentProfile.ts, suhaeng.ts
+ client.ts / envelope.ts / index.ts
```

### 3.1 패턴 (homework 예시)

```ts
import { apiClient } from './client'
import { unwrap, type ApiEnvelope } from './envelope'

export interface HomeworkResponse { id: number; courseId: number; title: string; ... }
export interface CreateHomeworkRequest { title: string; description?: string; dueAt?: string }

export const homeworkApi = {
  async listMine(): Promise<HomeworkResponse[]> {
    const res = await apiClient.get<ApiEnvelope<HomeworkResponse[]>>('/v1/students/me/homework')
    return unwrap(res.data)
  },

  async submit(homeworkId: number, content: string): Promise<SubmissionResponse> {
    const res = await apiClient.post<ApiEnvelope<SubmissionResponse>>(
      `/v1/students/me/homework/${homeworkId}/submit`,
      { content },
    )
    return unwrap(res.data)
  },

  // ... 강사용 ...
}
```

### 3.2 컨벤션

- 파일명 = 도메인명 (BE 패키지명과 1:1).
- export 는 `{domain}Api` 단일 object.
- 메서드 명명: `listMine / listByCourse / create / update / delete / submit / grade ...` 동사 기반.
- 모든 메서드가 `unwrap(res.data)` 호출 — envelope 풀어서 data 만 반환.
- 타입은 `Response / Request` suffix (예: `HomeworkResponse / CreateHomeworkRequest`).
- path 규칙: `/v1/students/me/...`, `/v1/teachers/me/...`, `/v1/academies/me/...`, `/v1/master/...` ([[../bkurs-be/auth-and-security#8.1]] 매칭).
- DELETE 응답은 `void` 반환 (`await apiClient.delete(...)` 만, 응답 본문 무시).

### 3.3 신규 도메인 client 추가 절차

1. BE 에 `{domain}` 패키지가 있는지 확인 ([[../bkurs-be/domain-catalog]]).
2. `src/lib/api/{domain}.ts` 생성.
3. Response / Request 타입 — BE response record 와 컬럼 1:1. `camelCase` 변환은 BE 가 응답 시점에 자동 (Jackson).
4. `{domain}Api` object — endpoint 1:1 메서드.
5. `src/lib/api/index.ts` 에 export 추가 (re-export).
6. `lib/hooks/use{Domain}.ts` 작성 (§4 참조).

## 4. `lib/hooks/use{Domain}.ts` — react-query wrapper

18 개. 도메인 client 의 메서드를 react-query `useQuery` / `useMutation` 으로 wrap.

### 4.1 패턴 (homework 예시)

```ts
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { homeworkApi, type HomeworkResponse, ... } from '@/lib/api/homework'

const HW_MINE_KEY = ['homework', 'mine'] as const
const HW_BY_COURSE_KEY = (cid: number) => ['homework', 'course', cid] as const
const HW_SUBMISSIONS_KEY = (hid: number) => ['homework', hid, 'submissions'] as const

export function useMyHomework() {
  return useQuery<HomeworkResponse[]>({
    queryKey: HW_MINE_KEY,
    queryFn: () => homeworkApi.listMine(),
  })
}

export function useSubmitHomework() {
  const qc = useQueryClient()
  return useMutation<SubmissionResponse, Error, { homeworkId: number; content: string }>({
    mutationFn: ({ homeworkId, content }) => homeworkApi.submit(homeworkId, content),
    onSuccess: (data) => {
      qc.setQueryData<SubmissionResponse[]>(MY_SUBS_KEY, (prev) => ...)
      qc.invalidateQueries({ queryKey: HW_MINE_KEY })
    },
  })
}
```

### 4.2 query key 컨벤션 (`decisions/decision-hook-convention`)

`as const` tuple — `[도메인, ...sub]`:

| 패턴 | 예 |
| --- | --- |
| 도메인 전체 | `['homework', 'mine']` |
| ID 기반 | `(cid: number) => ['homework', 'course', cid]` |
| 중첩 sub-resource | `(hid: number) => ['homework', hid, 'submissions']` |

규칙:
- 도메인 prefix 가 invalidate 의 단위.
- ID 인자가 있으면 함수 형태 (closure 로 key generator).
- `as const` 필수 — TypeScript narrowing.

### 4.3 onSuccess — invalidate / setQueryData

- 단순: `invalidateQueries({ queryKey: [...] })` 로 cache stale.
- 최적화: `setQueryData<T[]>(key, prev => updated)` 로 optimistic 패치 + invalidate 함께.
- mutation 의 onSuccess 가 같은 도메인의 list / detail 양쪽을 무효화하는지 확인.

### 4.4 enabled gate

`enabled: courseId > 0` 처럼 ID 조건. 0 / undefined 일 때 query 자동 disabled.

### 4.5 신규 hook 추가 절차

1. `lib/api/{domain}.ts` 가 먼저 존재해야 함.
2. `src/lib/hooks/use{Domain}.ts` 생성.
3. key 상수 정의.
4. read 는 `useQuery`, write 는 `useMutation`.
5. `lib/hooks/index.ts` 에 export 추가.
6. 페이지에서 hook 호출 + Loading / Error / Empty 처리.

## 5. `lib/auth/` — 인증 atom

### 5.1 `studentAuth.ts`

```ts
export const studentTokenAtom = atom<string | null>(...)            // localStorage sync
export const studentProfileAtom = atom<StudentProfile>(...)         // 학생 프로필
export const EMPTY_STUDENT_PROFILE: StudentProfile = ...
export type StudentRole = 'high_student' | 'middle_student'
```

학생 측만 — admin / master 는 각자 `_store/auth.ts` (페이지 한정).

### 5.2 `atoms.ts` — legacy shim

```ts
export const studentState = studentProfileAtom    // 기존 코드 호환용 alias
export const academyState = atom((get) => ...)    // derived
```

> ⚠️ supabase mock 단계의 import path 와 호환을 위한 임시 layer. concept 등 mock 페이지가 hook
> 으로 swap 되면 본 파일 제거 예정. 신규 코드는 `studentAuth.ts` 직접 사용.

## 6. `lib/utils/`

### `grade.ts`

학년 표시 / 라벨 helper. 예: `gradeLabel(1) → '1학년'`. 작은 util — 도메인 무관.

> util 이 도메인 한정으로 커지면 그 도메인의 api/hooks 옆에 두고 lib/utils 는 정말 공통만.

## 7. `lib/env.ts`

`VITE_*` 환경 변수 normalize. 누락 시 throw 또는 warn.

```ts
export const env = {
  API_BASE_URL: import.meta.env.VITE_API_BASE_URL ?? '/api',
  SUPABASE_URL: import.meta.env.VITE_SUPABASE_URL,
  SUPABASE_ANON_KEY: import.meta.env.VITE_SUPABASE_ANON_KEY,
} as const
```

## 8. `lib/http.ts` — legacy fetch wrapper

inroad-high 의 fetch wrapper. **점진 제거 대상** — 모든 호출이 `lib/api/client.ts` (axios) 로
이전됨. 새 코드는 axios 만.

## 9. `lib/supabase.ts` — legacy supabase client

inroad-high 의 supabase 직접 호출 잔재. mock 단계의 concept / book 등이 일부 의존 — BE swap 후
제거. 자세히 [[state-and-auth#supabase-removal]].

## 10. 도메인별 client / hook 매핑 표

| 도메인 (BE) | client | hook | 사용 페이지 |
| --- | --- | --- | --- |
| auth | auth.ts | useAuth (login/me/changePassword) | _shared/auth/*, master/admin/login |
| academy | academy.ts | useAcademy | admin/academy, student/connect |
| announcement | announcement.ts | useAnnouncement | admin/announcements, student/sidebar |
| studentProfile | studentProfile.ts | useStudentProfile | student/connect/signup, admin/students/detail |
| plan | plan.ts | usePlan | admin/billing, master/billing |
| course | (academy 또는 course 일부 — 점진) | (course hook 후속) | admin/teachers, high/middle 페이지 |
| homework | homework.ts | useHomework | middle/homework, admin/middle-tabs/homework |
| statement | statement.ts | useStatement | (후속) |
| question | question.ts | useQuestion | high/expect, middle/expect |
| pastQuestion | pastQuestion.ts | usePastQuestion | high/past, middle/past, master/questions |
| majorQuestion | majorQuestion.ts | useMajorQuestion | high/major, master/questions |
| interview | interview.ts | useInterview | high/mockexam (후속) |
| record | record.ts | useRecord | high/record |
| discussion | discussion.ts | useDiscussion | high/topic, middle/debate |
| reading | reading.ts | useReading | high/book, middle/book |
| performance | performance.ts | usePerformance | (admin 학생 상세) |
| simulation | simulation.ts | useSimulation | high/sim, middle/sim |
| search | search.ts | (hook 후속) | (글로벌 검색 UI) |
| stats | stats.ts | (hook 후속) | admin/students/detail, dashboard |
| dashboard | dashboard.ts | useDashboard | admin/dashboard, teacher (후속) |
| concept | concept.ts | useConceptDiagnosis | high/middle concept |
| saenggibu | saenggibu.ts | useSaenggibu | middle/record |
| suhaeng | suhaeng.ts | useSuhaeng | high/suhaeng, middle/suhaeng |

> 미완 항목 (course / search / stats hook 등) 은 [[../bkurs-fe/screen-logic-and-api-map]] 의 swap
> roadmap 에 따라 추가.

## cross-link

- [[bkurs-fe]]
- [[directory-structure]]
- [[architecture-api-foundation]] — envelope / apiClient 도입 ADR (#60)
- [[screen-logic-and-api-map]] — 화면별 hook + endpoint 매핑
- [[decisions/decision-hook-convention]] — query key 규칙
- [[../bkurs-be/common-module-reference]] — BE envelope 1:1 매칭
- [[../bkurs-be/auth-and-security]] — 401 / token 흐름 BE 측
