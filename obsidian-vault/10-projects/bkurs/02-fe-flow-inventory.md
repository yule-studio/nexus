---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, fe, inventory, mock, supabase]
related:
  - "[[00-index]]"
  - "[[01-project-map]]"
  - "[[03-be-api-contract]]"
  - "[[05-operational-risk-register]]"
---

# 02 · FE 데이터 흐름 인벤토리

> Source of truth: `/Users/masterway/masterway-dev/bkurs/bkurs-fe/src/router.tsx` + `src/pages/<role>/_pages/*` + `src/lib/api/*` + `src/lib/hooks/*` + `src/lib/supabase.ts`.
>
> Data source 분류 vocabulary:
> - **BE API** — `src/lib/api/*` 또는 `src/lib/hooks/*` 통해 bkurs-be 호출
> - **Supabase** — `import { supabase } from '@/lib/supabase'` 후 직접 호출
> - **mock** — `_data/` 의 정적 데이터 또는 모듈 내 상수
> - **localStorage** — `localStorage.setItem` / `getItem` 으로 클라 영속 (mock 동의어 — UI persistence)
> - **placeholder** — `alert(...)` / TODO / `S3 단계에서 ...` 주석으로 미구현 노출
> - **mixed** — 동일 페이지에서 2가지 이상 source 혼합

운영 배포 가능성 vocabulary (위험도 기반):
- **운영 OK** — BE/Supabase 호출 안정 + 회귀 테스트 / smoke 가능
- **운영 주의** — BE 호출 일부 + mock 일부 (UI 가능, 데이터 신뢰 X)
- **운영 보류** — mock / placeholder 가 핵심 — 실 운영 시 거짓 정보 노출 위험
- **cutover 필요** — Supabase 직접 호출 — BE 이전 결정 / 권한 분리 필요

---

## 1. master role

| Route | Page/Component | Data source | 관련 파일 | 운영 배포 가능성 | 후속 작업 |
| --- | --- | --- | --- | --- | --- |
| `/master/login` | `MasterLogin` | BE API | `src/pages/master/_pages/login/MasterLogin.tsx`, `useMasterLogin.ts`, `src/lib/api/auth.ts` | 운영 OK | role gate 검증 (MASTER 만) |
| `/master` (=`dashboard`) | `MasterDashboard` | mock | `src/pages/master/_pages/dashboard/MasterDashboard.tsx`, `dashboard-data.ts` | 운영 보류 | BE: `OwnerDashboardController` 와는 별개 — Master 전용 dashboard endpoint 부재 |
| `/master/academies`<br>`/master/academies/:id` | `MasterAcademies` / `MasterAcademyDetail` (+ tabs / modals) | mock + localStorage | `_pages/academies/academies-data.ts`, `academy-detail-data.ts`, `useAcademyMutations.ts` | 운영 보류 | BE: `AcademyController` 가 이미 있으므로 read 부분부터 swap 가능 |
| `/master/billing` | `MasterBilling` | mock | `_pages/billing/billing-data.ts`, `BillingPaymentsTable.tsx` (alert 포함) | 운영 보류 | BE: `PlanCatalog` / `Subscription` / `Payment` 도메인 활용 검토 |
| `/master/notices` | `MasterNotices` | localStorage (`master_notices`) | `useNotices.ts`, `notices-data.ts` | 운영 보류 | BE: `AnnouncementOwnerController` 가 학원 단위 — Master 통합 공지는 BE 신설 필요 |
| `/master/data` | `MasterData` | mock | `Data.tsx`, `data-page-data.ts`, `DataAiUsage.tsx` (alert) | 운영 보류 | BE: `OwnerStatsController` 일부 활용 가능 |
| `/master/audit` | `MasterAudit` | mock | `Audit.tsx`, `useAudit.ts` — `alert(...)` placeholder export | 운영 보류 | BE: audit log endpoint 신설 필요 |
| `/master/staff` | `MasterStaff` | localStorage (`master_staff`) | `useStaff.ts`, `staff-data.ts` | 운영 보류 | BE: Account 도메인 + Master 전용 invite 필요 |
| `/master/lessons` | `MasterLessons` | mock + Supabase placeholder | `useLessons.ts`, `useUploadFlow.ts`, `LessonUploadModal.tsx` | cutover 필요 | S3 cutover 시 `bkurs-be /api/v1/files` presigned + Supabase storage cleanup |
| `/master/questions` | `MasterQuestions` | **BE API** | `Questions.tsx` (`useMasterPastQuestions`, `useMasterMajorQuestions`) | **운영 OK** | 유일하게 BE 연동 완료 — 추후 passage tab 추가 |

> Master role 의 9/10 페이지가 mock 또는 localStorage 기반. 운영 노출 시 **데이터 신뢰성 0** — 즉 실 학원 / 매출 / 직원 / 감사 로그가 보이지 않는다. 운영 release 에서 임시 비활성 또는 read-only banner 가 안전. ([[05-operational-risk-register#R2]])

---

## 2. admin role (OWNER / TEACHER)

| Route | Page | Data source | 관련 파일 | 운영 배포 가능성 | 후속 작업 |
| --- | --- | --- | --- | --- | --- |
| `/admin/login` | `AdminLogin` | BE API | `useAdminLogin.ts`, `src/lib/api/auth.ts` | 운영 OK | role gate (OWNER/TEACHER) 검증 |
| `/admin` (=dashboard) | `AdminDashboard` | BE API | `AdminDashboard.tsx`, `useAdminDashboard` hook | 운영 OK | dashboard widget 별 endpoint 정합성 |
| `/admin/students`, `/:id` | `AdminStudents`, `StudentDetail` | mixed (BE + mock tabs) | `useStudents.ts`, `high-tabs/*`, `middle-tabs/*` | 운영 주의 | 일부 tab 이 `high-tabs-data` mock — record / homework / suhaeng 등 BE 도메인으로 swap |
| `/admin/middle-students`, `/:id` | (재사용 컴포넌트) | mixed | (동일) | 운영 주의 | router 가 `AdminStudents` 재사용 — middle 전용 필터링 검증 |
| `/admin/student-approval` | `StudentApproval` | BE API | `useStudentApproval.ts`, `src/lib/api/academy.ts` (OwnerMembershipQueue) | 운영 OK | |
| `/admin/teachers` | `Teachers` | BE API | `useTeachers.ts`, `academyApi.teachers`, `AddTeacherModal.tsx`, `DeleteTeacherModal.tsx` | 운영 OK | invite mail (BE NotificationService) mock 인 점 주의 |
| `/admin/academy` | `Academy` | BE API | `useMyAcademy.ts`, `myAcademyApi` | 운영 OK | |
| `/admin/billing` | `Billing` | BE API | `Billing.tsx`, `planApi` | 운영 주의 | PaymentGateway 가 mock — 실 결제 cutover 시 큰 변경 |
| `/admin/settings` | `Settings` | BE API + mock(일부) | `Settings.tsx` | 운영 주의 | school code / logo 등 일부 placeholder |
| `/admin/suhaeng` | `SuhaengManage` | BE API | `SuhaengManage.tsx`, `suhaengApi` | 운영 OK | |
| `/admin/announcements` | `AnnouncementManage` | BE API | `AnnouncementManage.tsx`, `announcementApi` | 운영 OK | |

특이사항:
- `AdminSidebar.tsx` 가 `supabase` 를 import 하지만 학원 로고 placeholder 용 — 실 호출 X. S3 cutover 시 cleanup. (`src/pages/admin/_layout/AdminSidebar.tsx:14`)
- `students/detail/high-tabs/SimulationTab.tsx`, `RecordTab.tsx`, `ExpectTab.tsx`, `middle-tabs/HomeworkTab.tsx`, `SuhaengTab.tsx` 등은 import 만 있고 호출은 mock — cleanup 대상.

---

## 3. high-student role

| Route | Page | Data source | 관련 파일 | 운영 가능성 | 후속 |
| --- | --- | --- | --- | --- | --- |
| `/high-student/login` | `Login` | BE API | `useStudentLogin.ts` | 운영 OK | |
| `/high-student/signup` | `Signup` | BE API | `useStudentSignup.ts` | 운영 OK | role 화이트리스트 (R1) 적용 후 |
| `/high-student/connect` | `Connect` | BE API | `useStudentConnect.ts`, `academyApi.connect` | 운영 OK | |
| `/high-student/pending` | `Pending` | BE API | `useStudentPending.ts` | 운영 OK | |
| `/high-student` (=`roadmap`) | `Roadmap` | mock | `Roadmap.tsx`, `_data/student-mock.ts` | 운영 주의 | BE roadmap endpoint 부재 — 신설 또는 dashboard 재사용 |
| `topic` | `TopicList` | BE API | `useStatement` / `useReading` hooks | 운영 OK | |
| `book` | `BookList` | BE API | `useReading`, `readingApi` | 운영 OK | |
| `record` | `Record` | BE API | `useRecord`, `recordApi` | 운영 OK | record entry source 분기 (B23) |
| `major` | `Major` | mock | `Major.tsx` | 운영 보류 | |
| `expect` | `Expect` | mock | `_data/expect-mock.ts`, `alert` 사용 | 운영 보류 | |
| `past` | `Past` | BE API | `usePastQuestion`, `pastQuestionApi` | 운영 OK | |
| `concept` | `CareerConcept` → `ConceptDiagnosis` / `ConceptResult` | **Supabase 직접** | `src/lib/supabase.ts` 호출 + `src/lib/hooks/useConceptDiagnosis.ts` (BE 계약 이미 보유) | **cutover 필요** | 페이지에 주석으로 cutover 계획 명시 (`CareerConcept.tsx` 최상단) |
| `presentation` | `Presentation` | mock | `Presentation.tsx`, `_data` | 운영 보류 | BE: discussion / interview 와 별개 — 신설 필요 |
| `mockexam` | `Mockexam` | mock | `Mockexam.tsx` | 운영 보류 | |
| `simulation` | `Simulation` | mixed (BE + placeholder) | `useSimulation`, `simulationApi` + 영상 placeholder | 운영 주의 | S3 storage cutover 후 video URL 부여 |
| `suhaeng` | `Suhaeng` | BE API | `useSuhaeng`, `suhaengApi` | 운영 OK | |

---

## 4. middle-student role

high-student 와 거의 같은 패턴. 차이 :

| Route | 변경 | Data source |
| --- | --- | --- |
| `lesson` | 추가 | mock (`Lesson.tsx`) |
| `debate` | 추가 (high 에는 없음) | BE API (`useDiscussion`, `discussionApi`) |
| `expect/wizard` | 추가 | mock (`EssayWizard.tsx`) |
| `homework` | 추가 (high 에는 별도 없음) | BE API (`useHomework`, `homeworkApi`) |
| `concept` | 동일 | **Supabase 직접** (`CareerConcept.tsx`, `ConceptResult.tsx`, `ConceptDiagnosis.tsx`) |

---

## 5. Supabase 직접 호출 파일 목록 (전수)

근거: `rg -l "from '@/lib/supabase'" src/`.

| 파일 | 사용 패턴 | cutover 우선순위 |
| --- | --- | --- |
| `src/lib/supabase.ts` | client 생성 (`createClient`) | base — 본 파일은 cutover 단계에서 삭제 또는 stub |
| `src/lib/env.ts` | `supabaseUrl` / `supabaseAnonKey` typed export | base |
| `src/lib/auth/atoms.ts` | session state atom — Supabase auth 동기화용 잔여? (확인 필요) | base — 검토 후 BE auth atom 으로 통일 |
| `src/lib/hooks/useConceptDiagnosis.ts` | BE concept endpoint 호출 hook (이미 BE 계약 보유) | n/a — Supabase 미사용, naming 만 일치 (확인 필요) |
| `src/pages/high-student/_pages/concept/CareerConcept.tsx` | `supabase.from('student_concept').select(...)` 직접 | **P0** — BE `StudentConceptController` 로 swap |
| `src/pages/high-student/_pages/concept/ConceptDiagnosis.tsx` | 동일 | **P0** |
| `src/pages/high-student/_pages/concept/ConceptResult.tsx` | 동일 | **P0** |
| `src/pages/middle-student/_pages/concept/CareerConcept.tsx` | 동일 | **P0** |
| `src/pages/middle-student/_pages/concept/ConceptDiagnosis.tsx` | 동일 | **P0** |
| `src/pages/middle-student/_pages/concept/ConceptResult.tsx` | 동일 | **P0** |
| `src/pages/master/_pages/lessons/useUploadFlow.ts` | 주석 — `supabase.storage.from('lessons').upload(...)` placeholder | **P1** — S3 cutover 시 같이 |
| `src/pages/admin/_layout/AdminLayout.tsx` | import 만 (학원 로고 placeholder) | cleanup |
| `src/pages/admin/_layout/AdminSidebar.tsx` | 동일 | cleanup |
| `src/pages/admin/_pages/suhaeng/SuhaengManage.tsx` | import 만 (실 호출 없음 추정) | cleanup 또는 미사용 import 제거 |
| `src/pages/admin/_pages/students/detail/middle-tabs/HomeworkTab.tsx` | mock + 주석 | cleanup |
| `src/pages/admin/_pages/students/detail/high-tabs/SimulationTab.tsx` | mock + 주석 (`S3 단계에서 supabase.storage signed URL ...`) | cleanup |
| `src/pages/admin/_pages/students/detail/high-tabs/ExpectTab.tsx` | 동일 | cleanup |
| `src/pages/admin/_pages/students/detail/high-tabs/RecordTab.tsx` | 동일 | cleanup |

> P0 cutover 대상은 6개 파일 (concept 도메인). BE 측은 `StudentConceptController` / `TeacherConceptController` 이미 존재 → FE 만 hook swap.

---

## 6. mock 데이터 / 파일 별도 목록

`_data/` 디렉터리 + `*-data.ts` / `*-mock*` 패턴.

| 파일 | 사용 페이지 | 영속 |
| --- | --- | --- |
| `src/pages/high-student/_data/student-mock.ts` | roadmap, expect, mockexam | 정적 |
| `src/pages/high-student/_data/student-mock2.ts` | (보조) | 정적 |
| `src/pages/high-student/_data/media-mock.ts` | presentation / simulation | 정적 |
| `src/pages/middle-student/_data/expect-mock.ts` | expect / wizard | 정적 |
| `src/pages/middle-student/_data/extras-mock.ts` | misc | 정적 |
| `src/pages/middle-student/_data/student-mock.ts` | roadmap 등 | 정적 |
| `src/pages/master/_pages/academies/academies-data.ts` | Academies / detail | 정적 |
| `src/pages/master/_pages/academies/detail/academy-detail-data.ts` | detail tabs | 정적 |
| `src/pages/master/_pages/billing/billing-data.ts` | Billing | 정적 |
| `src/pages/master/_pages/data/data-page-data.ts` | Data | 정적 |
| `src/pages/master/_pages/dashboard/...` (별도 mock) | Dashboard | 정적 |
| `src/pages/master/_pages/notices/notices-data.ts` | Notices | localStorage |
| `src/pages/master/_pages/staff/staff-data.ts` | Staff | localStorage |
| `src/pages/master/_pages/lessons/lessons-data.ts` | Lessons | 정적 + Supabase placeholder |
| `src/pages/admin/_pages/students/detail/high-tabs/high-tabs-data.ts` | StudentDetail high tabs | 정적 |
| `src/pages/admin/_pages/students/detail/middle-tabs/middle-tabs-data.ts` | StudentDetail middle tabs | 정적 |

---

## 7. FE 공식 데이터 흐름 — 정리 권고

운영 표면 정리 시 다음 순서를 권한다 (단정 아님):

1. **인증/세션 (P0)** — `apiClient` + `bkurs.accessToken` + refresh cookie 흐름. 이미 운영 OK 수준. Supabase atom 잔여 (`src/lib/auth/atoms.ts`) 점검 후 통일.
2. **학생/강사/원장 도메인 (P1)** — BE 22 client + 28 hook 이 이미 정착. 학생/admin 표면은 운영 OK or 주의 등급으로 묶을 수 있음.
3. **concept 도메인 cutover (P1)** — 6 파일 Supabase → BE `StudentConceptController` 로 hook swap. FE 변경 최소.
4. **Master role 정리 (P2)** — 9/10 페이지가 mock — 운영 release 에서 "internal preview" / "read-only" banner 또는 비활성 결정. BE Master 도메인 부재한 영역 (academies write, billing, staff, audit) 은 별도 endpoint 신설 후 swap.
5. **영상 / 파일 storage (P3)** — BE `FileStorageService` 의 S3 구현 + FE `useUploadFlow` swap. concept / record / saenggibu / lesson 모두 영향.

> 정확한 운영 조건은 [[05-operational-risk-register]] 와 [[06-deployment-readiness-roadmap]] 의 staging → prod 순서 따라 다시 평가.
