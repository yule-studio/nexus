---
title: "bkurs-fe — Role 모듈 인수인계"
kind: handover
status: stable
created_at: 2026-05-21T22:50:00+09:00
tags: [bkurs-fe, handover, roles, pages]
related:
  - "[[bkurs-fe]]"
  - "[[directory-structure]]"
  - "[[routing-and-protection]]"
  - "[[state-and-auth]]"
home_hub: bkurs-fe
---

# bkurs-fe — Role 모듈 인수인계

> 4 role (admin / master / high-student / middle-student) + `_shared` 의 책임 분담. 같은
> "학생 상세" 페이지가 admin 과 high-student 양쪽에 존재하는 이유, 공통 ThreadView 가 왜 양쪽
> `_components/` 에 따로 있는지 등.

## 0. role ↔ BE Role 매핑

| FE 경로 | BE Role enum | 사용자 |
| --- | --- | --- |
| `/master/*` | `MASTER` | 본사 운영 |
| `/admin/*` | `OWNER` (학원장) | 학원 운영 |
| `/high-student/*` | `STUDENT_HIGH` | 고등 학생 |
| `/middle-student/*` | `STUDENT_MIDDLE` | 중등 학생 |

> teacher (강사) role 의 FE 페이지는 현 sprint 범위 외 — BE 만 있음. 추후 추가될 가능성.

## 1. `pages/master/` — 본사 운영

**색상**: brand-master (보라 `#7C3AED`). [[design-system#brand-master]].

### 페이지 (9/9 완성)

| path | 컴포넌트 | 책임 |
| --- | --- | --- |
| `/master/login` | `MasterLogin` | 본사 계정 로그인 |
| `/master` (index) | `MasterDashboard` | 전체 학원 / 결제 / 사용자 통계 |
| `/master/academies` | `MasterAcademies` + `detail/AcademyDetail` | 학원 목록 + 상세 (탭: 사용자/구독/공지/감사 etc.) |
| `/master/billing` | `MasterBilling` | 본사 billing |
| `/master/notices` | `MasterNotices` | 본사 공지 |
| `/master/data` | `MasterData` | 데이터 export / xlsx |
| `/master/audit` | `MasterAudit` | 감사 로그 |
| `/master/staff` | `MasterStaff` | 본사 staff |
| `/master/lessons` | `MasterLessons` | 강의 콘텐츠 (전사) |
| `/master/questions` | `MasterQuestions` (default export) | 입시 문항 (Past / Major 탭) ([[domain-catalog#admission]] 매칭) |

### 인증 / 가드

- `MasterLayout` — `isMasterAuthenticatedAtom` 검사. 미인증 → `<Navigate to="/master/login" />`
- `_store/auth.ts` — `masterTokenAtom` + `isMasterAuthenticatedAtom`
- 로그인은 `/api/v1/auth/login` 호출 후 role=MASTER 만 통과

## 2. `pages/admin/` — 학원장 운영

**색상**: brand-primary (파랑 / 또는 brand-high). 학원장은 학생 (고등+중등) 양쪽 데이터 다 조회.

### 페이지 (8 + 학생 상세 19)

#### 보조 페이지 8개

| path | 컴포넌트 | 책임 |
| --- | --- | --- |
| `/admin/login` | `AdminLogin` | 학원장 로그인 |
| `/admin` (index) | `AdminDashboard` | 학원 9 지표 rollup |
| `/admin/academy` | `Academy` (admin) | 학원 정보 / 4 필드 |
| `/admin/billing` | `Billing` (admin) | 학원 결제 / 환불 / 구독 |
| `/admin/settings` | `Settings` (admin) | 학원 설정 |
| `/admin/teachers` | `Teachers` | 강사 invite + assignment |
| `/admin/student-approval` | `StudentApproval` | 학원 가입 대기 학생 승인 |
| `/admin/announcements` | `AnnouncementManage` | 공지 CRUD |
| `/admin/suhaeng` | `AdminSuhaengManage` | 수행/면담 관리 |

#### 학생 상세 19개 (`/admin/students/{id}`)

학원장이 학생을 클릭하면 그 학생의 모든 데이터를 보는 통합 페이지. 학생 group 에 따라 탭 다름:

- **고등 학생 (10 탭)**: `high-tabs/expect`, `high-tabs/past`, 기타 8 탭 (인라인)
- **중등 학생 (9 탭)**: `middle-tabs/middle-{expect,homework,past,presentation,suhaeng}` + 4 인라인

**핵심**: `high-tabs/expect/AiAnalysisPanel.tsx` 같은 component 는 학생 본인 페이지
(high-student/_pages/expect/) 와 **별도 instance**. 자세히 [[shared-components]].

### 인증 / 가드

- `AdminLayout` — `_store/auth.ts` 의 `isAdminAuthenticatedAtom`
- `legacy supabase` 의존 — 점진 제거 ([[state-and-auth]])

## 3. `pages/high-student/` — 고등 학생

**색상**: brand-high (파랑 `#2563EB`). [[design-system#brand-high]].

### 페이지 (13 + 부트스트랩 4 = 17)

#### 부트스트랩 4 (인증 흐름)

| path | 컴포넌트 | 책임 |
| --- | --- | --- |
| `/high-student/login` | `HighStudentLogin` | 로그인 — `_shared/auth/useStudentLogin('high')` |
| `/high-student/signup` | `HighStudentSignup` | 회원가입 — `useStudentSignup('high')` |
| `/high-student/connect` | `HighStudentConnect` | 학원 코드 입력 — `useStudentConnect('high')` |
| `/high-student/pending` | `HighStudentPending` | 학원장 승인 대기 — `useStudentPending('high')` |

#### 학습 페이지 13

| path | 컴포넌트 | 매핑 BE |
| --- | --- | --- |
| `/high-student` (index) `/roadmap` | `HighStudentRoadmap` | (FE only / inroad) |
| `topic` | `HighStudentTopicList` | discussion topic |
| `book` | `HighStudentBookList` | reading note |
| `record` | `HighStudentRecord` | record (생기부) |
| `major` | `HighStudentMajor` | admission/major-question |
| `expect` | `HighStudentExpect` | question (EXPECTED) — AiAnalysisPanel 사용 |
| `past` | `HighStudentPast` | admission/past-question — AiAnalysisPanel 사용 |
| `concept` | `HighStudentConcept` (`CareerConcept`) | concept |
| `presentation` | `HighStudentPresentation` | (FE) |
| `mockexam` | `HighStudentMockexam` | (FE / interview) |
| `simulation` | `HighStudentSimulation` | simulation |
| `suhaeng` | `HighStudentSuhaeng` | suhaeng |

### 공유 사이드

- `_layout/StudentLayout.tsx` — 사이드바 + Outlet + 인증 가드
- `_components/ThreadView.tsx` — 5 단계 답변 thread (expect / past / presentation 에서 사용)
- `_data/student-mock.ts`, `student-mock2.ts`, `media-mock.ts` — mock 데이터 (BE swap 후 제거 예정)

## 4. `pages/middle-student/` — 중등 학생

**색상**: brand-middle (초록 `#10B981`). [[design-system#brand-middle]].

### 페이지 (15 + 부트스트랩 4 = 19)

#### 부트스트랩 4 — high 와 동일 구조 (group='middle')

#### 학습 페이지 15

| path | 컴포넌트 | 매핑 BE / 비고 |
| --- | --- | --- |
| `/middle-student` (index) `/roadmap` | `MiddleStudentRoadmap` | |
| `lesson` | `MiddleStudentLesson` | course / lesson |
| `concept` | `MiddleStudentConcept` | concept |
| `book` | `MiddleStudentBookList` | reading note |
| `homework` | `MiddleStudentHomework` | homework |
| `expect` | `MiddleStudentExpect` | question (EXPECTED) — AiAnalysisPanel 사용 |
| `expect/wizard` | `MiddleStudentEssayWizard` | expect 의 sub-route |
| `past` | `MiddleStudentPast` | admission/past — AiAnalysisPanel + ScoreRadar |
| `presentation` | `MiddleStudentPresentation` | AiAnalysisPanel + ScoreRadar |
| `simulation` | `MiddleStudentSimulation` | simulation |
| `debate` | `MiddleStudentDebate` | discussion |
| `suhaeng` | `MiddleStudentSuhaeng` | suhaeng — AiAnalysisPanel 사용 |
| `record` | `MiddleStudentRecord` | saenggibu (중등 생기부) |

### 공유 사이드

- `_layout/StudentLayout.tsx` — middle 전용 사이드바
- `_components/ThreadView.tsx` — high 와 별도 instance ([[shared-components#thread-view]])
- `_data/student-mock.ts`, `expect-mock.ts`, `extras-mock.ts`

## 5. `pages/_shared/` — role 간 공용

```
_shared/auth/
├── useStudentLogin.ts      # group='high'|'middle' 인자 — 4 status (PENDING_CONNECT / PENDING_APPROVE / ACTIVE / REJECTED) 별 navigate 분기
├── useStudentSignup.ts
├── useStudentConnect.ts
└── useStudentPending.ts
```

핵심: **high / middle 양쪽이 동일 hook 4개를 group 인자로** 사용 — DRY. 자세히
[[state-and-auth#4-hook]].

## 6. cross-role 공통 vs role 한정 — 결정 기준

| 위치 | 의미 |
| --- | --- |
| `pages/{role}/_components/` | 본 role 내 1개 이상 페이지에서 재사용. cross-role 사용 없음. |
| `pages/_shared/` | 2개 이상 role 에서 사용하는 page-level helper (현재 auth 4 hook 만) |
| `components/` (top) | role 무관 layout / wrapper (`RootLayout`) |
| `lib/` | 비-React logic (api / hooks / auth atom / util) |

> ThreadView 가 high 와 middle 양쪽 `_components/` 에 **별도 존재하는 이유**: 디자인 변형
> (brand 색 / 사이드바 너비 / 텍스트 톤) 이 role 별로 다름. inroad-high 의 분기 그대로 보존.
> 만약 디자인 정확히 같으면 `_shared/` 로 승격 가능. 자세히 [[shared-components]].

## 7. 학생 상세 — admin 의 "고등/중등 탭"

`/admin/students/{studentId}` 의 `StudentDetail` 컴포넌트는 학생 group 판별 후:

- `students/detail/high-tabs/{expect,past}/` — 학생 본인 페이지에 있는 것과 **거의 동일한 컴포넌트**. AiAnalysisPanel 인라인.
- `students/detail/middle-tabs/{middle-expect,middle-homework,middle-past,middle-presentation,middle-suhaeng}/` — 중등 탭.

**왜 별도 디렉토리?** admin 의 학생 상세는 "본인이 아닌 학원장 시점에서 학생 자료를 본다" — 약간 다른 권한 / 추가 액션 (편집 권한 차이). inroad-high 의 패턴 그대로 옮김.

## 8. 신규 페이지 추가 절차

1. `pages/{role}/_pages/{feature}/` 폴더 — component 파일 + 필요 시 sub-component
2. `src/router.tsx` 의 해당 role children 배열에 `{ path: 'feature', element: <NewPage /> }` 추가
3. 데이터 fetch 는 `lib/hooks/use{Domain}.ts` 의 기존 hook 재사용 (없으면 신설 — [[lib-layer-guide]])
4. role 사이드바 (`_layout/{Role}Sidebar.tsx`) 에 메뉴 추가 — 일반적으로 별도 PR 또는 동일 PR 의 한 commit
5. 디자인 토큰 — `tailwind.config.js` 의 brand 만 사용. raw hex 직접 금지 ([[design-system]])
6. mock 데이터 의존 시 `_data/` 에 추가. BE swap 후 제거 — TODO 명시

## 9. role 간 import 금지 규칙

- `pages/high-student/**` 가 `pages/middle-student/**` import 금지 (반대도). cross-role 공유는 `_shared/` 또는 `lib/` 승격.
- `pages/admin/**` 가 `pages/high-student/**` 의 컴포넌트를 직접 import 하지 않는다 — 학생 상세는 admin 의 별도 인라인 컴포넌트.
- `_layout/` 끼리도 cross import 금지.

ESLint custom rule 으로 강제하면 좋지만 (후속), 현재는 PR review 로 catch.

## cross-link

- [[bkurs-fe]]
- [[directory-structure]] — 전체 src 지도
- [[routing-and-protection]] — router.tsx + 인증 가드
- [[shared-components]] — ThreadView / AiAnalysisPanel / ScoreRadar 매핑
- [[state-and-auth]] — Jotai atom + 4 status 흐름
- [[lib-layer-guide]] — api / hooks 컨벤션
- [[design-system]] — brand-high / middle / master 토큰
