---
title: bkurs-fe — 화면별 로직 / API 매핑
kind: reference
status: active
created_at: 2026-05-21
tags: [bkurs-fe, screen-map, api-map]
related:
  - "[[bkurs-fe]]"
  - "[[architecture-api-foundation]]"
  - "[[fe-domain-clients-complete]]"
  - "[[../bkurs-be/external-services-guide]]"
home_hub: "[[bkurs-fe]]"
---

# bkurs-fe — 화면별 로직 / API 매핑

## 1. 핵심 요약

bkurs-fe 의 모든 화면을 **사용자 그룹 × 화면 × 사용 hook × BE endpoint** 로 정리.
신규 개발자가 "이 화면은 무엇을 호출하나" 한 눈에 파악하기 위한 표.

> swap 진행 상황은 `status` 열 — **DONE** (BE 실데이터) / **MOCK** (정적 mock) /
> **N/A** (BE 도메인 미존재, 신설 필요).

| 사용자 | 화면 수 | DONE | MOCK | N/A |
|--------|--------|------|------|------|
| admin (학원장 + 강사) | 9 | 9 | 0 | 0 |
| high-student | 17 | 7 | 0 | 10 |
| middle-student | 16 | 8 | 0 | 8 |
| master (본사) | 10 | 1 | 0 | 9 |

(2026-05-21 기준 — 학생 onboarding 4개 + master/login 모두 DONE)

**그룹별 swap 진척**:
- admin **100%** swap 완료 (PR #69~#107 누적).
- high/middle-student **인증 / 학습 핵심 도메인** swap 완료 (login/signup/connect/pending +
  concept/topic/book/record/suhaeng/(homework)/(saenggibu)). 입시 mock 화면 (expect/past/
  mockexam/major/presentation/simulation/roadmap) 은 BE 입시 도메인 신설 후 swap 예정.
- master 는 **login 만 DONE**, 나머지 9개 화면은 BE master 도메인 신설 후 swap 예정.

---

## 2. admin (학원장 / 강사) — 9 화면

| 화면 | 경로 | 핵심 hook | BE endpoint | status |
|------|------|-----------|-------------|--------|
| 로그인 | `/admin/login` | `useLogin` | `POST /v1/auth/login` (role 검증 OWNER/TEACHER) | **DONE** |
| 대시보드 | `/admin` | `useOwnerDashboard`, `useOwnedStudents` | `GET /v1/academies/me/dashboard`, `GET /v1/academies/me/students` | **DONE** |
| 학생 관리 | `/admin/students` | `useOwnedStudents` | `GET /v1/academies/me/students` (email + profile join) | **DONE** |
| 학생 승인 큐 | `/admin/student-approval` | `usePendingMemberships`, `useApproveMembership`, `useRejectMembership` | `GET /v1/academies/me/memberships/pending`, `POST /v1/academies/memberships/{id}/approve`, `POST /v1/academies/memberships/{id}/reject` | **DONE** |
| 강사 관리 | `/admin/teachers` | `useOwnedTeachers`, `useTeacherInvites`, `useIssueTeacherInvite`, `useRemoveTeacher` | `GET /v1/academies/me/teachers`, `GET /v1/academies/me/teacher-invites`, `POST /v1/academies/me/teacher-invites`, `POST /v1/academies/me/teachers/{id}/remove` | **DONE** |
| 학원 코드 관리 | `/admin/academy` | `useMyAcademy`, `useAcademyCodes`, `useIssue/DeactivateAcademyCode` | `GET /v1/academies/me`, `GET /v1/academies/me/codes`, `POST /v1/academies/me/codes`, `POST /v1/academies/me/codes/{id}/deactivate` | **DONE** |
| 결제 관리 | `/admin/billing` | `useMySubscription`, `useMyPayments`, `useActivePlans` | `GET /v1/academies/me/subscription`, `GET /v1/academies/me/payments`, `GET /v1/plans` | **DONE** |
| 학원 공지 | `/admin/announcements` | `useOwnedAnnouncements`, `useCreate/Update/DeleteAnnouncement` | `GET /v1/academies/me/announcements`, `POST /v1/academies/me/announcements`, `PATCH /v1/academies/me/announcements/{id}`, `DELETE /v1/academies/me/announcements/{id}` | **DONE** |
| 수행평가 관리 | `/admin/suhaeng` | `useAcademySuhaengList`, `useCreateSuhaengForStudent`, `useFeedbackRound`, `useFinalizeRound` | `GET /v1/academies/{aid}/suhaeng`, `POST /v1/academies/{aid}/suhaeng`, `POST /v1/suhaeng/rounds/{rid}/feedback`, `POST /v1/suhaeng/rounds/{rid}/finalize` | **DONE** |
| 학원 설정 | `/admin/settings` | `useMyAcademy`, `useUpdateMyAcademy`, `useActivePlans`, `useMySubscription`, `useChangePassword` | `GET/PATCH /v1/academies/me`, `GET /v1/plans`, `GET /v1/academies/me/subscription`, `POST /v1/auth/change-password` | **DONE** |

**admin 영역 로직 패턴**:
- `academyState` jotai atom — 로그인 시 학원/role 정보 캐시 (legacy, 점진적으로 `useMyAcademy` 로 swap 중)
- 모든 mutation 은 `setQueryData` 또는 `invalidateQueries` 로 캐시 갱신
- 학생/강사 식별은 **email 기반** (Account 에 name 컬럼 없음)

---

## 3. high-student (고등 학생) — 17 화면

| 화면 | 경로 | 핵심 hook | BE endpoint | status |
|------|------|-----------|-------------|--------|
| 로그인 | `/high-student/login` | `useStudentLogin('high')` | `POST /v1/auth/login` (role 검증 STUDENT_HIGH) | **DONE** |
| 회원가입 | `/high-student/signup` | `useStudentSignup('high')` | `POST /v1/auth/signup` + `POST /v1/students/me/profile` (grade 4~6) | **DONE** |
| 학원 연결 | `/high-student/connect` | `useStudentConnect('high')` | `POST /v1/auth/connect` (학원 코드) | **DONE** |
| 가입 대기 | `/high-student/pending` | `useStudentPending('high')` (5s polling) | `GET /v1/students/me/membership` | **DONE** |
| 로드맵 | `/high-student/roadmap` | (mock data only) | — | N/A |
| 진로 진단 | `/high-student/concept` | `useConceptDiagnosis`, `useUpdateConceptProgress`, `useCompleteConcept` | `GET/PUT/POST /v1/students/me/concept-diagnosis` | **DONE** |
| 탐구주제 | `/high-student/topic` | `useMyDiscussions`, `useMyDiscussionPost`, `useSubmitPost` | `GET /v1/students/me/discussions`, `GET /v1/students/me/discussions/{tid}/my-post`, `POST /v1/students/me/discussions/{tid}/posts` | **DONE** |
| 독서 노트 | `/high-student/book` | `useMyReadingNotes`, `useCreate/Update/DeleteReadingNote`, `useMyNoteFeedback` | `GET /v1/students/me/reading-notes`, CRUD + `GET /.../{id}/feedback` | **DONE** |
| 생기부 | `/high-student/record` | `useMyRecords` | `GET /v1/students/me/records?category=...` (8 카테고리) | **DONE** |
| 수행평가 | `/high-student/suhaeng` | `useMySuhaengList`, `useSubmitSuhaeng`, `useResubmitSuhaeng` | `GET /v1/students/me/suhaeng`, `POST /v1/students/me/suhaeng/{id}/submit-first`, `POST /v1/students/me/suhaeng/{id}/resubmit` | **DONE** |
| 면접 시뮬 | `/high-student/simulation` | (mock — 영상 녹화 흐름) | BE simulation 도메인은 case-study (불일치) | N/A |
| 예상 질문 / 기출 / 모의면접 / 학과 / 제시문 | `/high-student/{expect,past,mockexam,major,presentation}` | (mock data only) | — | N/A — 입시 도메인 BE 신설 필요 |

**high-student 로직 패턴**:
- `studentProfileAtom` jotai — 로그인 시 학년/학교 캐시 (mock)
- 화면 진입 시 `useMyXxx()` 패턴 — `enabled` 옵션은 학생 ACTIVE 일 때만 (status guard)
- DEBATE 토픽일 때만 position (FOR/AGAINST/NEUTRAL) 토글 노출

---

## 4. middle-student (중등 학생) — 16 화면

| 화면 | 경로 | 핵심 hook | BE endpoint | status |
|------|------|-----------|-------------|--------|
| 로그인 | `/middle-student/login` | `useStudentLogin('middle')` | `POST /v1/auth/login` (role 검증 STUDENT_MIDDLE) | **DONE** |
| 회원가입 | `/middle-student/signup` | `useStudentSignup('middle')` | `POST /v1/auth/signup` + `POST /v1/students/me/profile` (grade 1~3) | **DONE** |
| 학원 연결 | `/middle-student/connect` | `useStudentConnect('middle')` | `POST /v1/auth/connect` | **DONE** |
| 가입 대기 | `/middle-student/pending` | `useStudentPending('middle')` | `GET /v1/students/me/membership` | **DONE** |
| 진로 진단 | `/middle-student/concept` | `useConceptDiagnosis` | 동일 | **DONE** |
| 토론 주제 | `/middle-student/debate` | `useMyDiscussions`, `useMyDiscussionPost`, `useSubmitPost` | (high-student/topic 와 동일) | **DONE** |
| 독서 노트 | `/middle-student/book` | `useMyReadingNotes`, CRUD | 동일 | **DONE** |
| 숙제 | `/middle-student/homework` | `useMyHomework`, `useMySubmissions`, `useSubmitHomework` | `GET /v1/students/me/homework`, `GET /v1/students/me/homework-submissions`, `POST /v1/students/me/homework/{id}/submit` | **DONE** |
| 수행평가 | `/middle-student/suhaeng` | (high-student/suhaeng 와 동일) | 동일 | **DONE** |
| 생기부 (saenggibu) | `/middle-student/record` | `useMySaenggibuTerm`, `useUpsertSaenggibu`, `useSaenggibuLock` | `GET /v1/students/me/saenggibu?grade=..&semester=..`, `PUT /v1/students/me/saenggibu`, `GET /v1/academies/{aid}/saenggibu-lock` | **DONE** |
| 수업 영상 | `/middle-student/lesson` | (mock — 영상 placeholder) | BE course 도메인은 session 단위 (영상 없음) | N/A |
| 예상 질문 / 기출 / 제시문 / 시뮬 | `/middle-student/{expect,past,presentation,simulation}` | (mock data only) | — | N/A |

**middle-student 차이점 (high 와)**:
- 생기부는 saenggibu 도메인 (학년/학기 단위 upsert) — high 의 record (학생 단위 entry) 와 다름
- 숙제 도메인은 중등만 사용
- 토론 디자인 — DEBATE 만 사용 (찬/반)

---

## 5. master (본사 super-admin) — 10 화면

| 화면 | 경로 | 핵심 hook | BE endpoint | status |
|------|------|-----------|-------------|--------|
| 로그인 | `/master/login` | `useMasterLogin` | `POST /v1/auth/login` (role=MASTER) | **DONE** |
| 그 외 9 화면 | — | (mock) | BE master 도메인 미존재 | **N/A** |

9 화면 BE 신설 시 별도 도메인 `master/*`:
- `/master/dashboard` — 전체 academy / 전체 학생 / 결제 합계
- `/master/academies` — 전체 학원 list + 본사 가입 승인
- `/master/billing` — 전체 결제 현황 + 미수금
- `/master/notices` — 본사 공지 (전 학원에 push)
- `/master/data` — 전사 데이터 / AI 사용량
- `/master/audit` — 시스템 audit log
- `/master/lessons` — 본사 강의 컨텐츠 (영상 업로드)
- `/master/questions` — 입시 기출/예상 질문 (학교 단위)
- `/master/staff` — 본사 직원 + 권한 매트릭스

> 입시 도메인 (questions/lessons) 은 high/middle student 의 입시 mock 화면들의 source.
> master/questions 신설 → student 측 expect/past 등 swap 가능.

---

## 6. 사용자 그룹별 진입 흐름

### 6.1. 학생 (high / middle)
1. signup (role=STUDENT_HIGH/MIDDLE) → status=PENDING_CONNECT
2. connect (학원 코드 입력) → membership PENDING_APPROVE
3. pending 대기 화면 → 학원장이 승인 시 status=ACTIVE
4. 진입 → 진로 진단 → roadmap → 도메인 화면들

### 6.2. 강사
1. owner 가 invite 토큰 발급
2. signup (role=TEACHER) 또는 기존 계정 + teacher-connect (토큰 입력)
3. admin 화면 진입 (담당 학생 list)

### 6.3. 학원장
1. signup (role=OWNER) → ACTIVE
2. 학원 생성 → 학원 코드 발급
3. 학생/강사 가입 승인 + 학원 운영

### 6.4. 마스터
1. 사전 등록 계정 (signup 외부 차단, DB 직접 또는 신설 endpoint)
2. master 화면 진입 (모든 학원 조회 / audit / 공지)

---

## 7. 화면 / hook 추가 시 체크리스트

새 BE 도메인 추가 후 FE 연동 단계:
1. `src/lib/api/<domain>.ts` — typed client (envelope unwrap)
2. `src/lib/hooks/use<Domain>.ts` — useQuery / useMutation
3. `src/lib/api/index.ts` + `src/lib/hooks/index.ts` re-export
4. 화면에서 `useXxx` import + 빈/로딩/에러 처리
5. 본 노트 §2-5 표에 행 추가
6. typecheck PASS + 통합 동작 확인

> swap 패턴은 `architecture-api-foundation.md` + `decision-hook-convention.md` 참조.

## 8. 관련 노트

- [[bkurs-fe]] — 프로젝트 허브
- [[architecture-api-foundation]] — ApiEnvelope unwrap 구조
- [[fe-domain-clients-complete]] — 18 도메인 client 매핑
- [[../bkurs-be/external-services-guide]] — BE 가 의존하는 외부 SaaS
- [[../bkurs-be/env-deployment-guide]] — env 배포

## 9. 참고

- 화면 SSoT: `bkurs-fe/src/pages/{admin,high-student,middle-student,master}/_pages/`
- hook SSoT: `bkurs-fe/src/lib/hooks/`
- client SSoT: `bkurs-fe/src/lib/api/`
- 라우팅: `bkurs-fe/src/router.tsx`
