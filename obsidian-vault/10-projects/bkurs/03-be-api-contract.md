---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, be, api, envelope, contract]
related:
  - "[[00-index]]"
  - "[[01-project-map]]"
  - "[[02-fe-flow-inventory]]"
  - "[[05-operational-risk-register]]"
---

# 03 · BE API 계약 / Envelope 적용 현황

> Source: `/Users/masterway/masterway-dev/bkurs/bkurs-be/src/main/java/com/bkurs/*/controller/*.java` + `common/api/ApiEnvelope.java` + `common/api/ApiEnvelopes.java` + `common/handler/GlobalExceptionHandler.java` + `common/config/OpenApiConfig.java`.
>
> **`ApiEnvelope<T>` invariant** (`src/main/java/com/bkurs/common/api/ApiEnvelope.java:34-41`)
> - `success=true` ⇒ `data != null && error == null`
> - `success=false` ⇒ `data == null && error != null`
> - `meta` 는 항상 필수 (`requestId`, `timestamp`)

본 문서는 confirmed 사실 + 일부 추측 (확인 필요) 을 구분해 정리한다.

---

## 1. Envelope 적용 정책 (확인 / 사실)

- 모든 데이터 응답은 `ResponseEntity<ApiEnvelope<T>>` 또는 `ResponseEntity<ApiEnvelope<Void>>` 로 wrapping. 헬퍼는 `ApiEnvelopes.ok(...)` / `created(...)` / `fail(...)` / `noContent()`.
- 예외 흐름은 `GlobalExceptionHandler` (`@Order(LOWEST_PRECEDENCE)`) 가 `BusinessException` / `MethodArgumentNotValidException` / `HttpMessageNotReadableException` / `AccessDeniedException` / `AuthenticationCredentialsNotFoundException` / 405 / 406 / 415 / 404 / 일반 Exception 을 envelope.fail 로 변환. `requestId` 는 `RequestIds.REQUEST_ID_ATTR` 에서 수집.
- 예외 envelope 의 `error` 형태:
  ```json
  {
    "code": "AUTH-4001",
    "message": "...",
    "status": 401,
    "messageKey": "error.auth.invalid_credentials",
    "fieldErrors": [ { "field":"email", "message":"..." } ]
  }
  ```
  FE 의 `ApiError` 타입 (`src/lib/api/envelope.ts:11-33`) 과 1:1 매칭.

### 1.1 envelope 미적용 (의도된 예외)

| Endpoint | 응답 형태 | 이유 |
| --- | --- | --- |
| `POST /api/v1/auth/logout` | `204 No Content` + Set-Cookie 만 | RFC 7231 — 204 는 body 없음. `AuthController.logout` (`controller/AuthController.java:114-121`) |
| `POST /api/v1/auth/change-password` | `204 No Content` | 동일 — `AuthController.changePassword:142-146` |
| `GET /api/v1/health` | `Map<String,Object>` 단순 ping | 게이트웨이/CDN 라우팅 검증 — envelope X 명시 (`common/health/HealthController.java`) |
| `/actuator/health`, `/actuator/info` | Spring Boot Actuator 기본 | management endpoint — envelope X |
| `GET /api/v1/students/me/record.pdf` 등 PDF blob | `application/pdf` 바이너리 | binary stream — envelope 으로 wrap 불가. `pdf/StudentRecordPdfController.java` |
| `POST /api/v1/files/upload-url` 자체는 envelope (✓) — 그러나 그 발급 URL 로 클라가 직접 PUT 하는 흐름은 BE 의 envelope 외부 | — | 외부 storage 직접 호출 |

### 1.2 권고 — envelope 예외 규칙 명문화 (제안)

운영 진입 전 다음을 코드 SSoT 또는 vault SSoT 로 명시:

1. **데이터 응답 ⇒ envelope 필수.** `ApiEnvelopes.ok(...)` / `created(...)` 만 사용. 직접 `ResponseEntity.ok(data)` 금지.
2. **side-effect 만 있고 데이터 없음 ⇒ 204 + envelope X.** logout / change-password / delete (단, 삭제 후 `success: true, data: null` 도 가능). 컨트롤러 javadoc 에 사유 명시.
3. **binary stream (PDF / image / csv) ⇒ envelope X + Content-Type 명시.** 본문 상단 javadoc 에 "envelope 미적용" 표기.
4. **health / metrics ⇒ envelope X.** OpenAPI 스펙에 명시.
5. **인증 redirect / OAuth callback (B28 후속) ⇒ envelope 미적용 가능.** 별도 결정 필요.

---

## 2. Endpoint 인벤토리 (도메인별)

> path prefix 는 `@RequestMapping` 의 단일 값. 메서드별 `@GetMapping` / `@PostMapping` 등은 본 인벤토리에서는 prefix 단위로만 정리 — 메서드 단위 표는 후속 PR 에서 보강.

> `Envelope` 컬럼: ✓ (적용) / **204** / **blob** / **ping** / **확인**.

> `FE client` 컬럼: `src/lib/api/*` 의 어떤 모듈이 대응하는지.

### 2.1 auth
| Method/Path | Controller | Response type | Envelope | FE client | 비고 |
| --- | --- | --- | --- | --- | --- |
| `POST /api/v1/auth/signup` | `AuthController` | `TokenResponse` | ✓ | `src/lib/api/auth.ts` | role 화이트리스트 미적용 (R1) |
| `POST /api/v1/auth/login` | 동 | `TokenResponse` | ✓ | 동 | |
| `POST /api/v1/auth/refresh` | 동 | `TokenResponse` | ✓ | `apiClient` 인터셉터 자체 호출 | |
| `POST /api/v1/auth/logout` | 동 | (없음) | **204** | `apiClient.logout()` | |
| `GET  /api/v1/auth/me` | 동 | `MeResponse` | ✓ | `authApi.me()` | |
| `POST /api/v1/auth/change-password` | 동 | (없음) | **204** | `authApi.changePassword()` | |
| `POST /api/v1/auth/connect` | `academy/ConnectController` | (학원 connect 결과) | ✓ | `useStudentConnect.ts`, `academyApi.connect` | path 가 `/auth` 하위에 있는 점은 의도 — 가입 직후 흐름 |
| `POST /api/v1/auth/teacher-connect` | `academy/TeacherConnectController` | (강사 connect) | ✓ | `academyApi.teacherConnect` | |

### 2.2 academy (13 controller)
| Path prefix | Controller | 책임 | FE client | Envelope |
| --- | --- | --- | --- | --- |
| `/api/v1/academies` | `AcademyController` | 학원 조회 (코드 기반) | `academyApi.getByCode` | ✓ |
| `/api/v1/academies/me` | `MyAcademyController` 외 다수 | 본인 학원 + 코드 + invite + transfer + queue | `myAcademyApi.*` | ✓ |
| `/api/v1/academies/my` | `MyAcademyController`, `StudentCourseController`, `AnnouncementMemberController` | 멤버 시각 (학생 / 강사) — 본인 소속 학원 보기 | `academyApi.my*`, `courseApi.my*`, `announcementApi.my*` | ✓ |
| `/api/v1/academies/memberships` | `MembershipController` | membership CRUD | `academyApi.membership*` | ✓ |
| `/api/v1/academies/me/codes` | `AcademyCodeController` | 학원 코드 발급 / 회전 | `academyApi.codes` | ✓ |
| `/api/v1/academies/me/teacher-invites` | `TeacherInviteController` | 강사 초대 | `academyApi.teacherInvite` | ✓ |
| `/api/v1/academies/me/owner-transfers` | `OwnerTransferFromController` | 원장 → 양도 발신 | `academyApi.ownerTransferFrom` | ✓ |
| `/api/v1/owner-transfers` | `OwnerTransferToController` | 양수 수신 | `academyApi.ownerTransferTo` | ✓ |
| `/api/v1/academies/me/memberships` | `OwnerMembershipQueueController` | pending 학생 승인 큐 | `useStudentApproval` | ✓ |

### 2.3 course (9 controller)
| Path | Controller | FE client |
| --- | --- | --- |
| `/api/v1/academies/me/courses` | `OwnerCourseController` | `useAcademyCourses` |
| `/api/v1/academies/my/courses` | `StudentCourseController` | `useMyCourses` (학생 시각) |
| `/api/v1/teachers/me/courses` | `TeacherCourseController` | `useTeacherCourses` |
| `/api/v1/teachers/me/courses` | `TeacherCourseUpdateController` | `useTeacherCourseUpdate` |
| `/api/v1/academies/me` (sessions) | `OwnerSessionController` | `useAcademySessions` |
| `/api/v1/academies/me/sessions` | `OwnerAttendanceController` | `useAcademyAttendance` |
| `/api/v1/teachers/me` (attendance) | `TeacherSessionAttendanceController` | `useTeacherAttendance` |
| `/api/v1/students/me` (attendance) | `StudentSessionAttendanceController` | `useStudentAttendance` |
| `/api/v1/teachers/me/students` | `TeacherStudentProfileController` | `useTeacherStudents` |

### 2.4 학습 도메인 (record / statement / question / discussion / reading / performance / interview / simulation / suhaeng / saenggibu / concept / homework)

각 도메인은 학생 + 강사 (+ 일부 원장) 컨트롤러로 분리. 모두 `ApiEnvelope` 적용 — `controller/*.java` 의 `ResponseEntity<ApiEnvelope<...>>` signature 로 확인.

| 도메인 | 학생 prefix | 강사 prefix | (옵션) 원장 prefix | FE client |
| --- | --- | --- | --- | --- |
| record | `/students/me` | `/teachers/me` | `/academies/me` | `recordApi`, `useRecord` |
| statement | `/students/me/personal-statements` | `/teachers/me` | — | `statementApi`, `useStatement` |
| question | `/students/me` | `/teachers/me` | — | `questionApi`, `useQuestion` |
| discussion | `/students/me` | `/teachers/me` | — | `discussionApi`, `useDiscussion` |
| reading | `/students/me/reading-notes` | `/teachers/me` | — | `readingApi`, `useReading` |
| performance | `/students/me` | `/teachers/me` | — | `performanceApi`, `usePerformance` |
| interview | `/students/me` | `/teachers/me` | — | `interviewApi`, `useInterview` |
| simulation | `/students/me` | `/teachers/me` | — | `simulationApi`, `useSimulation` |
| suhaeng | `/students/me/suhaeng` | `/teachers/...` | `/academies/...` | `suhaengApi`, `useSuhaeng` |
| saenggibu | `/students/me/saenggibu` | `/teachers/academies/{id}/saenggibu` | `/academies/{id}/saenggibu/lock` | `saenggibuApi`, `useSaenggibu` |
| concept | `/students/me/concept-diagnosis` | `/teachers/me` | — | `conceptApi`, `useConceptDiagnosis` (**FE 일부는 아직 Supabase 직접**) |
| homework | `/students/me` | `/teachers/me` | — | `homeworkApi`, `useHomework` |
| admission/past-question | `/students/me/past-questions` | — | `/master/past-questions` | `pastQuestionApi`, `usePastQuestion` |
| admission/major-question | `/students/me/major-questions` | — | `/master/major-questions` | `majorQuestionApi`, `useMajorQuestion` |

### 2.5 보조 도메인
| 도메인 | path | controller | FE client | Envelope |
| --- | --- | --- | --- | --- |
| announcement (member view) | `/api/v1/academies/my/announcements` | `AnnouncementMemberController` | `announcementApi.my*` | ✓ |
| announcement (owner write) | `/api/v1/academies/me/announcements` | `AnnouncementOwnerController` | `announcementApi.me*` | ✓ |
| dashboard | `/api/v1/teachers/me`, `/api/v1/academies/me` | `TeacherDashboardController`, `OwnerDashboardController` | `dashboardApi`, `useAdminDashboard` | ✓ |
| stats | `/api/v1/students/me`, `/api/v1/teachers/me`, `/api/v1/academies/me` | `*StatsController` | `statsApi` | ✓ |
| search | `/api/v1/students/me`, `/api/v1/teachers/me`, `/api/v1/academies/me` | `*SearchController` | `searchApi` | ✓ |
| files | `/api/v1/files/upload-url` | `FileStorageController` | `fileApi` | ✓ (URL 발급만. 본 발급 후 클라가 외부 storage 로 직접 PUT) |
| pdf | `/api/v1/.../record.pdf` | `StudentRecordPdfController` | `recordPdfApi`, `saveBlobAsFile` | **blob** |
| plan (catalog) | `/api/v1/plans` | `PlanCatalogController` | `planApi.plans` | ✓ |
| plan (admin write) | `/api/v1/admin/plans` | `AdminPlanController` | `planApi.adminPlans` | ✓ |
| subscription | `/api/v1/academies/me/subscription` | `SubscriptionController` | `planApi.subscription` | ✓ |
| payment | (서비스 호출) | `PaymentController` | `planApi.payment` | ✓ |
| health | `/api/v1/health` | `HealthController` | (FE 호출 없음 — gateway 용) | **ping** |

### 2.6 보안 통과 (permitAll)
`SecurityConfig.filterChain` 의 permitAll:
- `/api/v1/auth/signup`, `/login`, `/refresh`, `/logout`
- `/api/v1/health`, `/actuator/health`, `/actuator/info`
- `/swagger-ui.html`, `/swagger-ui/**`, `/v3/api-docs`, `/v3/api-docs/**`, `/swagger-resources/**`

그 외 모든 endpoint 는 `authenticated()`. `@EnableMethodSecurity` 활성화 → 컨트롤러별 `@PreAuthorize` 적용 가능 (`FileStorageController` 이 `@PreAuthorize("isAuthenticated()")` 예시).

> permitAll 의 swagger 노출 위험은 [[05-operational-risk-register#R5]].

---

## 3. FE/BE 계약 정합성 — 빠른 일치 확인

| FE client (`src/lib/api/*`) | BE controller | 정합성 |
| --- | --- | --- |
| `auth.ts` | `AuthController` | ✓ 일치 |
| `academy.ts` | `AcademyController` 외 11 | ✓ 일치 (코드 / membership / invite / transfer / queue) |
| `pastQuestion.ts`, `majorQuestion.ts` | `admission/*` | ✓ 일치 (학생 / master 양쪽) |
| `studentProfile.ts` | `StudentProfileController`, `OwnerStudentProfileController`, `TeacherStudentProfileController` | ✓ 일치 |
| `concept.ts` | `concept/*Controller` | ✓ 일치 — 그러나 **FE 학생 페이지가 hook 대신 supabase 직접 호출** ([[02-fe-flow-inventory#5]]) |
| `suhaeng.ts` | `suhaeng/*` | ✓ |
| `saenggibu.ts` | `saenggibu/*` | ✓ |
| `homework.ts` | `homework/*` | ✓ |
| `question.ts`, `discussion.ts`, `reading.ts`, `interview.ts`, `simulation.ts`, `record.ts`, `statement.ts`, `performance.ts` | 각 `*Controller` | ✓ |
| `dashboard.ts`, `stats.ts`, `search.ts`, `announcement.ts` | 동일 | ✓ |
| `plan.ts` | `PlanCatalogController`, `SubscriptionController`, `AdminPlanController`, `PaymentController` | ✓ |
| `file.ts` | `FileStorageController`, PDF | ✓ |
| `session.ts` | (FE 인증 atom 헬퍼 — BE 직접 매핑 없음) | n/a |

### mismatch / 주의

1. **concept** — FE 학생 페이지는 `conceptApi` 가 아닌 `supabase` 사용. 인증 / 권한 / row-level secret 분리 위험. cutover 가 가장 적은 변경으로 가능 ([[02-fe-flow-inventory#5]]).
2. **Master role 의 academies / billing / staff / audit / notices / data** — FE 가 mock — BE 측 매핑 endpoint 부재. Master 가 운영 필요한 데이터는 별도 BE 도메인 신설 결정 필요 ([[07-open-questions#Q3]]).
3. **Health** — FE 는 직접 호출하지 않지만 BE 의 envelope 미적용 점은 의도. OpenAPI 스펙 / runbook 에 명시 권고.
4. **PDF blob** — FE `recordPdfApi` + `saveBlobAsFile` — envelope 외부. 권한 검증은 BE 가 책임. 다운로드 URL 의 토큰 만료 정책 확인 필요 (별도 PR).

---

## 4. 공통 error response shape (확인)

성공:
```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "meta": { "requestId": "req-...", "timestamp": "2026-05-31T12:34:56Z", "durationMs": 12 }
}
```

실패 (`GlobalExceptionHandler` 변환):
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "AUTH-4001",
    "message": "...",
    "status": 401,
    "messageKey": "error.auth.invalid_credentials",
    "fieldErrors": [
      { "field": "email", "message": "이메일 형식이 아닙니다" }
    ]
  },
  "meta": { "requestId": "req-...", "timestamp": "..." }
}
```

FE 의 `ApiClientError` (`src/lib/api/envelope.ts:73-90`) 가 envelope.error 를 그대로 throw. 화면 단에서 `error.code` / `error.messageKey` 로 분기.

---

## 5. ApiEnvelope 표준 — 제안 규칙 (요약)

운영 진입 전 vault 또는 BE README 에 단일 문장으로 박아두는 것을 권한다:

1. **모든 데이터 응답은 `ApiEnvelope<T>` 로 감싼다.** `ResponseEntity.ok(data)` 직접 호출 금지.
2. **side-effect 만 있고 응답이 비어 있으면 204 No Content + Set-Cookie / Location 사용.** envelope 부재 사유는 javadoc 에 한 줄.
3. **binary stream (PDF / image / csv) 은 envelope 외부.** `Content-Type` / `Content-Disposition` 책임은 컨트롤러.
4. **health / metrics 는 envelope 외부, 인증 외부.**
5. **모든 예외는 `BusinessException` 으로 던지고 `ErrorCodeSpec.status()` 로 매핑.** 글로벌 핸들러에서 envelope.fail 자동 변환.
6. **fieldErrors[]** 는 `@Valid` 위반에 한해 채운다. 도메인 규칙 위반은 `code` + `messageKey` 로 표현.

이 규칙들은 코드 SSoT 가 이미 80% 강제 (`ApiEnvelope` invariant + `GlobalExceptionHandler`). 나머지 20% 는 컨트롤러 javadoc + PR 리뷰로 강제.
