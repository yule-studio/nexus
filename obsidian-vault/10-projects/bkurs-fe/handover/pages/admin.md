---
title: "bkurs-fe pages — admin (학원장)"
kind: handover
status: stable
created_at: 2026-05-22T01:25:00+09:00
tags: [bkurs-fe, handover, pages, admin]
related:
  - "[[catalog]]"
  - "[[../../bkurs-be/handover/auth-and-security]]"
home_hub: "[[../../bkurs-fe]]"
---

# admin (학원장) 페이지

> BE Role: `OWNER`. brand 색: brand-high `#2563EB` (파랑). 학생 (고등+중등) 양쪽 데이터 통합 조회.
> 보조 페이지 8 + 학생 상세 19 + 공지 관리 = ~28 화면.

## 1. 디렉토리

```
pages/admin/
├── _layout/AdminLayout.tsx       # 인증 가드 + 사이드바
├── _store/auth.ts                # adminTokenAtom + academyAtom + isAdminAuthenticatedAtom
└── _pages/
    ├── login/AdminLogin.tsx
    ├── dashboard/AdminDashboard.tsx
    ├── academy/Academy.tsx
    ├── billing/Billing.tsx
    ├── settings/Settings.tsx
    ├── teachers/Teachers.tsx
    ├── student-approval/StudentApproval.tsx
    ├── announcements/AnnouncementManage.tsx
    ├── suhaeng/SuhaengManage.tsx
    └── students/
        ├── AdminStudents.tsx
        └── detail/
            ├── StudentDetail.tsx              # 학생 group 판별 후 high/middle 탭
            ├── high-tabs/
            │   ├── expect/                     # AiAnalysisPanel inline
            │   └── past/                       # AiAnalysisPanel inline
            └── middle-tabs/
                ├── middle-expect/
                ├── middle-homework/
                ├── middle-past/                # AiAnalysisPanel + ScoreRadar
                ├── middle-presentation/        # AiAnalysisPanel + ScoreRadar
                └── middle-suhaeng/             # AiAnalysisPanel
```

## 2. 페이지 ↔ hook ↔ BE endpoint

### 보조 페이지 8

| path | 컴포넌트 | hook (lib/hooks) | BE endpoint (lib/api) |
| --- | --- | --- | --- |
| `/admin/login` | `AdminLogin` | useLogin | `POST /api/v1/auth/login` ([[../api-clients/auth]]) |
| `/admin` (index) | `AdminDashboard` | useDashboard | `GET /v1/academies/me/dashboard` ([[../api-clients/dashboard]]) |
| `/admin/academy` | `Academy` | useAcademy.useMyAcademy | `GET /v1/academies/me` ([[../api-clients/academy]]) |
| `/admin/billing` | `Billing` | usePlan + usePayment (후속) | `/v1/academies/me/subscription`, `/payments` ([[../api-clients/plan]]) |
| `/admin/settings` | `Settings` | useAcademy.usePatchMine | `PATCH /v1/academies/me` |
| `/admin/teachers` | `Teachers` | useTeacherInvite / useTeacherAssignment | `/v1/academies/me/teacher-invites`, `/teachers` |
| `/admin/student-approval` | `StudentApproval` | useMembership.useQueue / useApprove / useReject | `/v1/academies/me/memberships/pending`, `/approve`, `/reject` |
| `/admin/announcements` | `AnnouncementManage` | useAnnouncement | `/v1/academies/me/announcements` ([[../api-clients/announcement]]) |
| `/admin/suhaeng` | `AdminSuhaengManage` | useSuhaeng | `/v1/academies/{id}/suhaeng` ([[../api-clients/suhaeng]]) |

### 학생 상세 (`/admin/students/{id}`)

`StudentDetail.tsx` 가 group 판별 후 high/middle 탭 렌더링.

| 탭 (고등) | hook | BE endpoint |
| --- | --- | --- |
| 학생 프로필 | useStudentProfile.useByOwner | `GET /v1/academies/me/students/{id}/profile` ([[../api-clients/studentProfile]]) |
| record (생기부) | useRecord.useByOwner | `GET /v1/academies/me/students/{id}/records` ([[../api-clients/record]]) |
| stats | useStats (후속) | `GET /v1/academies/me/students/{id}/stats` |
| search | useSearch (후속) | `GET /v1/academies/me/students/{id}/search` |
| expect 탭 (`high-tabs/expect/`) | usePastQuestion / useQuestion (후속) | `/students/me/past-questions` 등 |
| past 탭 | usePastQuestion | |

| 탭 (중등) | hook | 비고 |
| --- | --- | --- |
| middle-expect | (mock 후 swap) | AiAnalysisPanel |
| middle-homework | useHomework | `/v1/teachers/me/homework/{id}/submissions` 등 |
| middle-past | usePastQuestion | + ScoreRadar |
| middle-presentation | (mock) | + ScoreRadar |
| middle-suhaeng | useSuhaeng | AiAnalysisPanel |

## 3. 인증 / 가드

`AdminLayout` 가 `isAdminAuthenticatedAtom` 검증 → false 시 `<Navigate to="/admin/login" />`.

`_store/auth.ts` 의 atom:
```ts
adminTokenAtom        : atomWithStorage<AdminToken>('adminToken', ...)
academyAtom           : atomWithStorage<Academy>('admin.academy', ...)
isAdminAuthenticatedAtom: derived (token + expiresIn > now)
```

## 4. 공유 컴포넌트 사용

- `AiAnalysisPanel` — 6회 (4 middle 탭 + 2 high 탭)
- `ScoreRadar` — 2회 (middle-past / middle-presentation)
- 자세히 → [[../modules/shared-components]]

## 5. cross-link

- [[catalog]] — 4 role 인덱스
- [[../architecture/routing-and-protection]] — router 가드
- [[../architecture/state-and-auth]] — adminTokenAtom 상세
- [[../api-clients/catalog]] — 22 client 인덱스
- [[../../bkurs-be/handover/domains/academy]] — OWNER 측 BE 도메인
