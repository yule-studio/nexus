---
title: "bkurs-be — 디렉토리 구조 인수인계"
kind: handover
status: stable
created_at: 2026-05-21T22:00:00+09:00
tags: [bkurs-be, handover, structure, backend]
related:
  - "[[../bkurs-be]]"
  - "[[catalog]]"
  - "[[overview]]"
  - "[[common]]"
home_hub: bkurs-be
---

# bkurs-be — 디렉토리 구조 인수인계

> 신규 합류자 1순위 가이드. **"어디에 뭐가 있는가"** 를 한 장에. 도메인 내부 규칙은
> [[catalog]] 와 [[overview]] 로 위임.

## 1. 레포 최상단

```
bkurs-be/
├── .env.example              # 로컬 환경 변수 템플릿 (실제값은 .env.local — gitignore)
├── .github/workflows/        # CI (Gradle + JDK21 + MySQL service) + image publish
├── build.gradle              # Spring Boot 3.5 / Java 21 / dependency 선언
├── settings.gradle           # 단일 모듈 (rootProject.name = 'bkurs-be')
├── gradlew / gradlew.bat     # Gradle Wrapper (팀 표준)
├── Dockerfile                # multi-stage build (Eclipse Temurin 21)
├── docker-compose.yml        # prod-like compose (app + mysql)
├── docker-compose.dev.yml    # 로컬 mysql 만 띄우는 lightweight compose
├── docs/
│   ├── BRANCH_STRATEGY.md    # main / dev / release/* / feature/* / chore/* git-flow
│   ├── COMMIT_CONVENTION.md  # 한국어 + gitmoji + 3 섹션
│   └── CHANGELOG.md          # 사이클별 누적 (B1 → Phase C → 사이클 2)
├── AGENTS.md                 # 에이전트 / 사람 공용 진입점
├── CLAUDE.md                 # Claude Code 작업 시 hard rail
├── README.md                 # 빠른 시작 (Java/DB/실행/스크립트)
└── src/                      # 본 문서 §2~§4
```

## 2. `src/main/java/com/bkurs/` — 도메인 패키지 23

**규칙: 1 도메인 = 1 패키지.** 도메인 간 직접 참조 최소화 — 필요 시 service 호출, repository
직접 참조는 cross-domain validation (예: `AcademyMembershipResolver`) 한정.

| 패키지 | 역할 한 줄 | 자세히 |
| --- | --- | --- |
| `auth` | 회원가입 / 로그인 / Refresh / JWT / Account | [[catalog#auth]], [[auth-and-security]] |
| `academy` | 학원 + 회원/강사 가입·승인 + 학원장 이관 + 강사 invite | [[catalog#academy]] |
| `announcement` | 학원 공지 + 읽음 추적 | [[catalog#announcement]] |
| `student` | 학생 프로필 (school / grade / parentPhone …) | [[catalog#student]] |
| `plan` | 플랜 카탈로그 + 구독 + 결제 + PortOne 게이트웨이 | [[catalog#plan]] |
| `course` | 수업 + Session + Attendance | [[catalog#course]] |
| `homework` | 숙제 + 제출 + 채점 | [[catalog#homework]] |
| `statement` | 자소서 (DRAFT / SUBMITTED) + feedback | [[catalog#statement]] |
| `question` | 예상/기출 문항 + 응답 | [[catalog#question]] |
| `interview` | 모의면접 + audio/video URL 응답 + 평가 | [[catalog#interview]] |
| `record` | 학생부 8 카테고리 + 4 도메인 자동 bridge | [[catalog#record]] |
| `discussion` | 토론 토픽 + post (upsert) + 협업 평가 | [[catalog#discussion]] |
| `reading` | 독서 노트 + 교사 feedback | [[catalog#reading]] |
| `performance` | 수행평가 + 제출 (재제출 채점 무효화) | [[catalog#performance]] |
| `simulation` | 모의지원 시나리오 + feasibility 1~5 | [[catalog#simulation]] |
| `search` | 5 도메인 통합 LIKE 검색 + 권한 분기 | [[catalog#search]] |
| `stats` | 학생 7 영역 통계 aggregation | [[catalog#stats]] |
| `dashboard` | 강사 (수업별) / 학원장 (학원 rollup) 대시보드 | [[catalog#dashboard]] |
| `concept` | 진로 / 컨셉 진단 | [[catalog#concept]] |
| `suhaeng` | 중등 수행/면담 round 기록 | [[catalog#suhaeng]] |
| `saenggibu` | 중등 생기부 작성 (semester lock) | [[catalog#saenggibu]] |
| `admission` | 과거 / 전공 면접 문항 (master CRUD + student view) | [[catalog#admission]] |
| `file` | 파일 storage gateway (presigned URL) | [[external-gateways#file]] |
| `pdf` | PDF 생성 gateway (학생부) | [[external-gateways#pdf]] |
| `notification` | email/sms gateway (SendGrid) | [[external-gateways#notification]] |
| `ai` | Claude AI gateway | [[external-gateways#ai]] |
| `stt` | Clova STT gateway | [[external-gateways#stt]] |
| `batch` | Spring Batch job 정의 + cron launcher | [[overview#batch]] |
| `common` | envelope / error / handler / web / messaging / config / health | [[common]] |

## 3. 도메인 패키지 내부 — 7 sub-layer

거의 모든 도메인이 다음 sub-layer 구조를 동일하게 따른다.

```
{domain}/
├── controller/      # @RestController — HTTP entry. 권한 분기 (Student/Teacher/Owner/Master).
├── service/         # @Service — 트랜잭션 경계 + 비즈니스 invariant.
├── domain/          # @Entity + Repository — JPA + Spring Data.
├── dto/
│   ├── request/     # @Valid Bean Validation 대상 (CreateXRequest, UpdateXRequest …).
│   └── response/    # API 응답 record. entity 그대로 노출 금지.
├── enums/           # 도메인 상수 + 도메인 ErrorCode (예: AcademyErrorCode).
├── exception/       # BusinessException 서브타입 (예: AcademyNotFoundException).
└── handler/         # @RestControllerAdvice — basePackages = "com.bkurs.{domain}"
                     #  도메인 예외 → ApiEnvelope.fail 매핑.
```

**예외 패키지가 일부 도메인에 더 있음:**
- `gateway/` — 외부 SaaS 추상화 (file / notification / pdf / ai / stt / plan/payment). [[external-gateways]] 참조.
- `bridge/` — record 도메인 한정. 4 도메인 이벤트 → record auto-upsert.
- `jwt/` — auth 도메인 한정. token provider / filter / properties.
- `payment/` — plan 도메인 한정. PaymentGateway 구현 + props.

**규칙:**
- controller 는 service 호출 + envelope wrap (`ApiEnvelopes.ok(...)`) 만. 비즈니스 로직 금지.
- service 는 `@Transactional` 명시. read-only 는 `(readOnly = true)`.
- entity 는 setter 최소화 — `domain method` (예: `Academy.transferTo(...)`) 로 invariant 보호.
- DTO 는 record. request 는 `@Valid`, response 는 entity 가 아닌 record.
- 도메인 enum 의 ErrorCode 는 prefix 충돌 금지. e.g. `ACA-4040` (Academy), `PAY-4020` (Payment).
- 도메인 advice (`@RestControllerAdvice(basePackages = "com.bkurs.{domain}")`) 는 본인 도메인만. fallback 은 [[common#GlobalExceptionHandler]] 가 처리.

## 4. controller 명명 규칙 — role prefix

73 controller. **path role prefix 와 1:1 매핑.**

| prefix | path | 예시 |
| --- | --- | --- |
| `Student*` | `/v1/students/me/...` | `StudentHomeworkController`, `StudentRecordController` |
| `Teacher*` | `/v1/teachers/me/...` | `TeacherCourseController`, `TeacherStatsController` |
| `Owner*` | `/v1/academies/me/...` (학원장 본인 학원) | `OwnerCourseController`, `OwnerDashboardController` |
| `Master*` | `/v1/master/...` (본사 관리) | `MasterPlanController`, `MasterMajorQuestionController` |
| `Admin*` | `/v1/admin/...` (master 별칭, 일부) | `AdminPlanController` |
| `Auth*` / `Health*` | 무 prefix 또는 `/api/v1/auth`, `/api/v1/health` | `AuthController`, `HealthController` |

> 한 도메인에 student / teacher / owner 셋이 다 필요하면 controller 를 **3 개로 쪼개고** path
> prefix 로 권한 명확화. `@PreAuthorize` 도 같은 prefix 와 1:1.

## 5. `src/main/resources/`

```
resources/
├── application.yml           # 공통 (port / actuator / jwt secret env / SPRING_PROFILES_ACTIVE)
├── application-local.yml     # 로컬 (mysql 9 + Flyway validate + Hibernate format_sql)
├── application-dev.yml       # dev 환경 (배포된 mysql + log level INFO + CORS prod domain)
├── application-prod.yml      # production (env 변수 강제 + Flyway baseline-on-migrate=false)
└── db/migration/
    ├── V1__init_schema.sql          # placeholder
    ├── V2__create_account.sql       # auth
    ├── V3__create_refresh_token.sql # auth
    └── ... V29 까지 (자세히 [[database-and-migrations]])
```

profile 활성화: `SPRING_PROFILES_ACTIVE=local` (default) / `dev` / `prod`. local 은 `.env.local` 의
env 주입 (DB_PASSWORD, JWT_SECRET 등).

## 6. `src/test/`

51 test class. 거의 다 통합 테스트 (Repository + Service + Controller MockMvc).

```
src/test/java/com/bkurs/
├── AbstractIntegrationTest.java   # 모든 통합 테스트 공통 base — DB clean + JWT 발급 helper
├── auth/AuthApiTest.java
├── academy/AcademyApprovalApiTest.java
├── ...                            # 도메인별 *ApiTest.java
└── common/                        # envelope / handler unit 테스트
```

자세한 회귀 추가 절차는 [[testing-guide]].

## 7. 빠른 탐색 cheat sheet

| 찾고 싶은 것 | 어디로 |
| --- | --- |
| "이 URL 어디서 처리되지?" | `controller/` 에서 `@*Mapping("...")` grep |
| "이 비즈니스 룰 어디 있지?" | `service/` 에서 invariant 메시지 grep — 또는 [[catalog]] decision 링크 |
| "이 컬럼 추가 이력?" | `db/migration/V*.sql` grep, [[database-and-migrations]] 표 참조 |
| "이 에러 코드 (ACA-4040) 뭐지?" | `{domain}/enums/{Domain}ErrorCode.java` |
| "외부 SaaS swap?" | `{domain}/gateway/` + [[external-gateways]] |
| "공통 envelope / 401 / 403 / 405 처리?" | `common/handler/GlobalExceptionHandler.java` + [[common]] |

## cross-link

- [[../bkurs-be]] — 프로젝트 hub
- [[catalog]] — 23 도메인 상세
- [[overview]] — 요청 흐름 + 비동기 + batch
- [[common]] — common/* 모듈 사용법
- [[../../../20-areas/backend/spring/java-spring/java-spring]] — Spring 일반
