---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, fe, be, map]
related:
  - "[[00-index]]"
  - "[[02-fe-flow-inventory]]"
  - "[[03-be-api-contract]]"
---

# 01 · 프로젝트 지도 (FE / BE 큰 그림)

## 1. 레포 구조 요약

### bkurs-fe
근거: `/Users/masterway/masterway-dev/bkurs/bkurs-fe/src/router.tsx`,
`src/lib/`, `src/pages/`, `.env.example`, `.github/workflows/ci.yml`.

```
bkurs-fe/
├─ src/
│  ├─ router.tsx            # 4 role 라우트 단일 진입 (createBrowserRouter)
│  ├─ App.tsx, main.tsx
│  ├─ api/                  # (legacy / 빈 폴더, 본 분석에서 비활성)
│  ├─ lib/
│  │  ├─ env.ts             # import.meta.env typed wrapper
│  │  ├─ supabase.ts        # Supabase client (학생 concept / 일부 placeholder)
│  │  ├─ auth/              # 세션 atom + SessionBridge
│  │  ├─ api/               # bkurs-be REST 22 client (envelope unwrap)
│  │  └─ hooks/             # 28 TanStack Query / Mutation hook
│  ├─ pages/
│  │  ├─ _shared/auth/      # role 공용 학생 login / signup / connect / pending hook
│  │  ├─ common/            # HomePage / NotFoundPage
│  │  ├─ master/            # 본사 (10 page)
│  │  ├─ admin/             # 학원 admin (OWNER/TEACHER 공용)
│  │  ├─ high-student/      # 고등 학생
│  │  └─ middle-student/    # 중등 학생
│  ├─ components/, hooks/, store/, styles/, types/
├─ .env.example             # VITE_SUPABASE_*, VITE_API_BASE_URL, VITE_BE_ORIGIN, VITE_MOCK_STUDENT
└─ .github/workflows/ci.yml # lint + format + typecheck + build (test job 없음)
```

핵심 결정:
- **단일 SPA + role prefix 라우팅**. `/master/...`, `/admin/...`, `/high-student/...`, `/middle-student/...` (router.tsx:79~196).
- **API 호출은 `apiClient` (axios) — `withCredentials: true` + Bearer `bkurs.accessToken`** (`src/lib/api/client.ts`). 401 발생 시 단일 in-flight refresh 후 1회 재시도. 실패하면 `notifyUnauthorized()` 로 세션 정리.
- **응답 envelope `ApiEnvelope<T>` + `unwrap` / `unwrapNullable` 헬퍼** (`src/lib/api/envelope.ts`). BE 의 동명 record 와 1:1.
- **Supabase client 는 별도 존재** (`src/lib/supabase.ts`). 후속 cutover 대상 (concept 도메인 / 영상 placeholder).

### bkurs-be
근거: `/Users/masterway/masterway-dev/bkurs/bkurs-be/src/main/java/com/bkurs/`,
`src/main/resources/application*.yml`, `Dockerfile`, `docker-compose*.yml`, `.github/workflows/`.

```
bkurs-be/
├─ src/main/java/com/bkurs/
│  ├─ common/
│  │  ├─ api/               # ApiEnvelope, ApiMeta, ApiEnvelopes (helper)
│  │  ├─ handler/           # GlobalExceptionHandler / GlobalValidationExceptionHandler
│  │  ├─ config/            # SecurityConfig, CorsConfig, OpenApiConfig, AsyncConfig, TimeConfig
│  │  ├─ enums/, error/, exception/
│  │  ├─ health/            # /api/v1/health (Map<String,Object>, NOT envelope — 단순 ping)
│  │  ├─ messaging/         # RabbitConfig + RabbitMessagingPublisher / InMemory swap
│  │  └─ web/               # RequestIds 등 servlet helper
│  ├─ auth/                 # 인증 도메인 (signup/login/refresh/logout/me)
│  ├─ academy/              # 학원 + Connect + Membership + Owner/Teacher transfer
│  ├─ student/, course/, homework/, record/, statement/, question/, discussion/
│  ├─ reading/, performance/, interview/, simulation/, suhaeng/, saenggibu/
│  ├─ concept/              # 학생 진로 진단 (BE 보유 — FE는 아직 Supabase 일부 잔류)
│  ├─ admission/            # past-question / major-question
│  ├─ announcement/, dashboard/, stats/, search/
│  ├─ file/                 # FileStorageController + gateway (mock / S3)
│  ├─ pdf/                  # StudentRecordPdfController + PdfGenerator (mock)
│  ├─ plan/                 # PlanCatalog / Subscription / Payment + gateway (mock / PortOne)
│  ├─ notification/         # gateway (mock / SendGrid) + AsyncDispatcher
│  ├─ ai/                   # gateway (mock / Claude)
│  ├─ stt/                  # gateway (mock / Clova)
│  └─ batch/                # Spring Batch 잡 (OverduePaymentAlert 등)
├─ src/main/resources/
│  ├─ application.yml       # 공통 (jwt, cors, management.endpoints)
│  ├─ application-local.yml # 로컬 (DB localhost, mock 게이트웨이 default)
│  ├─ application-dev.yml   # staging (env 강제 ${...:?}, secure cookie ON, mock 게이트웨이 placeholder)
│  ├─ application-prod.yml  # prod (SSL 강제, validate 만, file logging)
│  └─ db/migration/V1..V29  # Flyway
├─ Dockerfile               # multi-stage temurin 21 jdk → jre slim, 비root user
├─ docker-compose.yml       # MySQL 9.0 + bkurs-mysql-data volume
├─ docker-compose.dev.yml   # 위 compose 위에 bkurs-be dev profile + mock 게이트웨이 전부 ON
└─ .github/workflows/
   ├─ ci.yml                # build + test (MySQL service container)
   └─ deploy-dev.yml        # GHCR image build only (실 deploy step 주석)
```

핵심 결정:
- **`/api/v1/{role}/me/...` 라우팅 컨벤션** — students/me, teachers/me, academies/me, admin, master, plans, files, auth.
- **`ApiEnvelope<T>` invariant** (`success/data/error/meta`) — 모든 컨트롤러가 `ApiEnvelopes` 헬퍼로 wrapping. 예외는 `GlobalExceptionHandler` 가 envelope.fail.
- **Spring Security 6 stateless** — JWT Bearer + HttpOnly refresh cookie. `permitAll` 은 signup/login/refresh/logout/health/swagger.
- **외부 SaaS는 gateway 인터페이스 + `@ConditionalOnProperty` 스왑** — `bkurs.external.{payment,file-storage,notification,ai,stt,pdf,mq}.impl=...`.
- **Flyway `validate` + `validate-on-migrate=true`** — JPA entity vs schema mismatch 시 startup fail.

## 2. FE role 별 화면 구조

근거: `src/router.tsx`, `src/pages/<role>/_pages/`.

### master (본사)
| 경로 | 컴포넌트 | 데이터 소스 | 참고 |
| --- | --- | --- | --- |
| `/master/login` | `MasterLogin` | BE auth (`/lib/api/auth.ts`) | role gate (`useMasterLogin`) |
| `/master` `index → dashboard` | `MasterDashboard` | 미상 (별도 mock) | dashboard-data 확인 필요 |
| `/master/academies` `[/:id]` | `MasterAcademies` / `MasterAcademyDetail` | `academies-data` (mock) + `useAcademyMutations` (mock) | tabs, modals 다수 |
| `/master/billing` | `MasterBilling` | `billing-data` (mock + localStorage) | matplotlib 대비 chart sub-cmp |
| `/master/notices` | `MasterNotices` | `useNotices()` localStorage `master_notices` | |
| `/master/data` | `MasterData` | `data-page-data` (mock) | |
| `/master/audit` | `MasterAudit` | `useAudit()` mock | `alert(...)` placeholder export |
| `/master/staff` | `MasterStaff` | `useStaff()` localStorage `master_staff` | |
| `/master/lessons` | `MasterLessons` | `useLessons()` + `useUploadFlow` (Supabase storage placeholder) | |
| `/master/questions` | `MasterQuestions` | BE: `usePastQuestion` / `useMajorQuestion` hooks | **유일하게 BE 연동 완료** |

### admin (학원 OWNER / TEACHER 공용)
| 경로 | 컴포넌트 | 데이터 소스 | 참고 |
| --- | --- | --- | --- |
| `/admin/login` | `AdminLogin` | BE auth | `useAdminLogin` |
| `/admin` | `AdminDashboard` | BE: `useAdminDashboard` hook (`/lib/api/dashboard.ts`) | |
| `/admin/students[/:id]` | `AdminStudents` + tabs | 일부 BE (`/lib/api/academy.ts` ownedStudents) + 일부 mock (`high-tabs-data`, `middle-tabs-data`) | tabs 가 도메인별 source 다양 |
| `/admin/middle-students` | (동일 컴포넌트 재사용) | 동일 | router 81 |
| `/admin/student-approval` | `StudentApproval` | BE: `useStudentApproval` (Owner membership queue) | |
| `/admin/teachers` | `Teachers` | BE: `academyApi` teacher endpoint | invite modal 포함 |
| `/admin/academy` | `Academy` | BE: `myAcademyApi` (`/lib/api/academy.ts`) | |
| `/admin/billing` | `Billing` | BE: `planApi` | |
| `/admin/settings` | `Settings` | BE: `myAcademyApi` 일부 | |
| `/admin/suhaeng` | `SuhaengManage` | BE: `suhaengApi` | |
| `/admin/announcements` | `AnnouncementManage` | BE: `announcementApi` | |

### high-student
| 경로 | 컴포넌트 | 데이터 소스 |
| --- | --- | --- |
| `/high-student/login` `signup` `connect` `pending` | role 공용 hook 사용 | BE auth + academy connect |
| `/high-student` `index → roadmap` | `HighStudentRoadmap` | mock (roadmap data alert TODO 일부 포함) |
| `topic` | `HighStudentTopicList` | BE: `useStatement` / `useReading` 등 |
| `book` | `HighStudentBookList` | BE: `useReading` |
| `record` | `HighStudentRecord` | BE: `useRecord` |
| `major` | `HighStudentMajor` | mock |
| `expect` | `HighStudentExpect` | mock (`_data/expect-mock`) |
| `past` | `HighStudentPast` | BE: `usePastQuestion` |
| `concept` | `HighStudentConcept` (CareerConcept) | **Supabase 직접 호출 (cutover 대기)** |
| `presentation` | `HighStudentPresentation` | mock + `_data` |
| `mockexam` | `HighStudentMockexam` | mock |
| `simulation` | `HighStudentSimulation` | BE: `useSimulation` + 일부 placeholder |
| `suhaeng` | `HighStudentSuhaeng` | BE: `useSuhaeng` |

### middle-student
- 동일 패턴. 추가: `lesson` (mock), `debate` (BE: `useDiscussion`), `expect/wizard` (mock).

> 정확한 호출 매핑은 [[02-fe-flow-inventory]] 표.

## 3. BE 도메인 패키지 구조

근거: `src/main/java/com/bkurs/*/controller/*.java` + `@RequestMapping`.

| 도메인 | controller 개수 | 대표 경로 prefix | 비고 |
| --- | --- | --- | --- |
| auth | 1 | `/api/v1/auth` | signup/login/refresh/logout/me/change-password |
| academy | 13 | `/api/v1/academies`, `/api/v1/academies/me`, `/api/v1/academies/my`, `/api/v1/auth` (connect) | 학원 + Connect + Membership + Owner/Teacher transfer + invite/code |
| course | 9 | `/api/v1/academies/me`, `/api/v1/teachers/me/courses`, `/api/v1/academies/my/courses`, `/api/v1/students/me` | course + session + attendance + student profile (teacher view) |
| record | 3 | `/api/v1/students/me`, `/api/v1/teachers/me`, `/api/v1/academies/me` | 학생 기록부 |
| statement | 2 | `/api/v1/students/me/personal-statements`, `/api/v1/teachers/me` | 자소서 + 피드백 |
| question | 2 | `/api/v1/students/me`, `/api/v1/teachers/me` | 문제 풀이 |
| discussion | 2 | `/api/v1/students/me`, `/api/v1/teachers/me` | 토론 |
| reading | 2 | `/api/v1/students/me/reading-notes`, `/api/v1/teachers/me` | 독서 노트 |
| performance | 2 | 동일 | 수행 평가 |
| interview | 2 | 동일 | 모의 면접 |
| simulation | 2 | 동일 | 시뮬레이션 |
| suhaeng | 3 | `/api/v1/students/me/suhaeng`, `/api/v1/teachers`, `/api/v1/academies` | 수행 평가 통합 |
| saenggibu | 3 | `/api/v1/students/me/saenggibu`, `/api/v1/teachers/academies/{id}/saenggibu`, `/api/v1/academies/{id}/saenggibu/lock` | 생기부 + lock |
| concept | 2 | `/api/v1/students/me/concept-diagnosis`, `/api/v1/teachers/me` | 진로 진단 |
| admission | 4 | `/api/v1/students/me/past-questions`, `/api/v1/master/past-questions`, `/api/v1/students/me/major-questions`, `/api/v1/master/major-questions` | 기출 / 학과 질문 |
| homework | 2 | `/api/v1/students/me`, `/api/v1/teachers/me` | 숙제 |
| dashboard | 2 | `/api/v1/teachers/me`, `/api/v1/academies/me` | 대시보드 |
| stats | 3 | `/api/v1/students/me`, `/api/v1/teachers/me`, `/api/v1/academies/me` | 통계 |
| search | 3 | 동일 | 통합 검색 |
| announcement | 2 | `/api/v1/academies/me/announcements`, `/api/v1/academies/my/announcements` | 공지 작성 / 조회 |
| plan | 4 | `/api/v1/plans`, `/api/v1/admin/plans`, `/api/v1/academies/me/subscription`, payment | 플랜 + 구독 + 결제 |
| file | 1 | `/api/v1/files` | presigned URL 발급 |
| pdf | 1 | `/api/v1` (record.pdf 등) | record PDF |
| common/health | 1 | `/api/v1/health` | envelope 미적용 (단순 ping) |

전체 endpoint 인벤토리는 [[03-be-api-contract]].

## 4. 큰 흐름도 (Mermaid)

### 4.1 인증 흐름
```mermaid
sequenceDiagram
  autonumber
  participant U as User (FE)
  participant FE as apiClient (axios)
  participant BE as AuthController
  participant DB as RefreshToken table

  U->>FE: 로그인 폼 제출
  FE->>BE: POST /api/v1/auth/login (email, pw)
  BE->>DB: insert refresh row (rotation chain)
  BE-->>FE: ApiEnvelope { data: {accessToken, account}, meta }
  BE-->>FE: Set-Cookie: bkurs_rt=...; HttpOnly; Secure(prod); SameSite=Lax
  FE->>FE: localStorage.setItem('bkurs.accessToken', token)

  Note over FE,BE: 이후 모든 호출 — Authorization: Bearer <token> + cookie 동봉
  FE->>BE: GET /api/v1/students/me/...
  BE-->>FE: 401 (만료)
  FE->>BE: POST /api/v1/auth/refresh (cookie 만)
  BE->>DB: validate + rotate (재사용 감지)
  BE-->>FE: new access + new cookie
  FE->>BE: 원 요청 1회 재시도 (Authorization 교체)
```

### 4.2 권한 분리 / 라우팅
```mermaid
flowchart TD
  subgraph FE[ bkurs-fe SPA ]
    Router[ router.tsx ]
    Router -->|/master/*| Master[ Master pages — mock + localStorage ]
    Router -->|/admin/*| Admin[ Admin pages — BE API + 일부 mock ]
    Router -->|/high-student/*| High[ High student pages — BE API + Supabase concept ]
    Router -->|/middle-student/*| Mid[ Middle student pages — BE API + Supabase concept ]
  end

  subgraph BE[ bkurs-be (Spring Boot 3.5) ]
    Auth[ AuthController /api/v1/auth ]
    Domain[ 60+ domain controllers /api/v1/{students|teachers|academies|admin|master}/me/... ]
    Health[ /api/v1/health (envelope X) ]
    Swagger[ /swagger-ui.html /v3/api-docs (permitAll) ]
  end

  Master --> Auth
  Master -.->|usePastQuestion/useMajorQuestion 만| Domain
  Admin --> Auth
  Admin --> Domain
  High --> Auth
  High --> Domain
  Mid --> Auth
  Mid --> Domain
  High -.->|concept 직접 호출| Supabase[(Supabase Postgres + Storage)]
  Mid -.->|concept 직접 호출| Supabase
  Master -.->|영상 storage placeholder| Supabase

  BE -.->|gateway @ConditionalOnProperty| Ext[(외부 SaaS — S3 / PortOne / SendGrid / Claude / Clova / RabbitMQ)]
```

### 4.3 외부 SaaS gateway 스왑
```mermaid
flowchart LR
  subgraph BE[ bkurs-be runtime ]
    PG[ PaymentGateway impl ]
    FS[ FileStorageService impl ]
    NS[ NotificationService impl ]
    AI[ AiAssistant impl ]
    STT[ SttService impl ]
    PDF[ PdfGenerator impl ]
    MQ[ MessagingPublisher impl ]
  end

  PG -. mock default .-> MockPay[ MockPaymentGateway @Component ]
  PG -. portone .-> PortOne[ PortOnePaymentGateway @ConditionalOnProperty ]
  FS -. mock default .-> MockFS[ MockFileStorage matchIfMissing ]
  FS -. s3 .-> S3FS[ (B17 — 미구현 placeholder) ]
  NS -. mock default .-> MockN[ MockNotificationService ]
  NS -. sendgrid .-> SG[ SendGridNotificationService ]
  AI -. mock default .-> MockAi[ MockAiAssistant ]
  AI -. anthropic .-> Claude[ ClaudeAiAssistant ]
  STT -. mock default .-> MockSTT[ MockSttService ]
  STT -. clova .-> Clova[ ClovaSttService ]
  PDF -. mock default .-> MockPDF[ MockPdfGenerator ]
  MQ -. in-memory .-> InMem[ InMemoryMessagingPublisher ]
  MQ -. rabbit .-> Rab[ RabbitMessagingPublisher ]
```

> 단, **MockPaymentGateway 만 `@ConditionalOnProperty` 없이 `@Component`** — `bkurs.external.payment.impl=portone` 일 때 양쪽 bean 등록 → `NoUniqueBeanDefinitionException` 위험. ([[05-operational-risk-register#R4]])
