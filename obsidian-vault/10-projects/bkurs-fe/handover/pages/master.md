---
title: "bkurs-fe pages — master (본사)"
kind: handover
status: stable
created_at: 2026-05-22T01:26:00+09:00
tags: [bkurs-fe, handover, pages, master]
related:
  - "[[catalog]]"
home_hub: "[[../../bkurs-fe]]"
---

# master (본사) 페이지

> BE Role: `MASTER`. brand 색: brand-master `#7C3AED` (보라). 본사 운영 — 9 페이지.

## 1. 디렉토리

```
pages/master/
├── _layout/MasterLayout.tsx        # 인증 가드 + MasterSidebar
├── _store/auth.ts                  # masterTokenState + masterState + isMasterAuthenticatedAtom
└── _pages/
    ├── login/MasterLogin.tsx
    ├── dashboard/MasterDashboard.tsx
    ├── academies/
    │   ├── Academies.tsx
    │   └── detail/AcademyDetail.tsx       # 학원별 상세 (tabs / modals)
    ├── billing/Billing.tsx
    ├── notices/Notices.tsx
    ├── data/Data.tsx
    ├── audit/Audit.tsx
    ├── staff/Staff.tsx
    ├── lessons/Lessons.tsx
    └── questions/Questions.tsx             # Past + Major 탭 (PR #132~#137 활성화)
```

## 2. 페이지 ↔ hook ↔ BE endpoint

| path | 컴포넌트 | hook | BE endpoint |
| --- | --- | --- | --- |
| `/master/login` | `MasterLogin` | useLogin | `POST /api/v1/auth/login` ([[../api-clients/auth]]) |
| `/master` (index) | `MasterDashboard` | (mock) | (후속 — 전사 통계) |
| `/master/academies` | `MasterAcademies` | (후속 — useMasterAcademies) | `/v1/master/academies` (후속) |
| `/master/academies/{id}` | `MasterAcademyDetail` | (후속) | `/v1/master/academies/{id}` |
| `/master/billing` | `MasterBilling` | usePlan.useAdminPlans / usePayments | `/v1/admin/plans`, `/admin/payments` ([[../api-clients/plan]]) |
| `/master/notices` | `MasterNotices` | (후속) | (후속 — 본사 공지) |
| `/master/data` | `MasterData` | (mock) | (xlsx export) |
| `/master/audit` | `MasterAudit` | (mock) | (후속 — audit log) |
| `/master/staff` | `MasterStaff` | (mock) | (후속) |
| `/master/lessons` | `MasterLessons` | (mock) | (강의 콘텐츠) |
| `/master/questions` | `MasterQuestions` | useMasterPastQuestions + useMasterMajorQuestions ([[../api-clients/pastQuestion]], [[../api-clients/majorQuestion]]) | `/v1/master/past-questions`, `/major-questions` |

## 3. 인증 / 가드

```ts
// _store/auth.ts
masterTokenState        : atomWithStorage<MasterToken>('masterToken', ...)
masterState             : atomWithStorage<Master>('masterInfo', DEFAULT_MASTER)
isMasterAuthenticatedAtom: atom((get) => {
    const t = get(masterTokenState)
    return !!t.accessToken && !!t.expiresIn && new Date(t.expiresIn).getTime() > Date.now()
})
```

`MasterLayout` 가드 — `<Navigate to="/master/login" replace />`.

## 4. 활성화 시리즈 (Questions PR #132~#137)

- 기존: master/questions placeholder + 11 broken file
- PR #128: named → default import 정정
- PR #132 (admission past-question): MASTER PastQuestion CRUD client + hook + tab 활성
- PR #134: 학생 측 swap (high/middle past)
- PR #136 (major-question): MajorQuestion CRUD + 양측 swap (#137)

## 5. cross-link

- [[catalog]]
- [[../architecture/routing-and-protection]]
- [[../architecture/state-and-auth#master]]
- [[../api-clients/pastQuestion]] / [[../api-clients/majorQuestion]]
- [[../../bkurs-be/handover/domains/admission]] — MASTER 발행 입시 문항 BE
