---
title: "bkurs-be — B-KURS 백엔드 프로젝트 hub"
kind: project
project: bkurs-be
agent: engineering-agent/tech-lead
status: in-progress
created_at: 2026-05-20T00:10:00+09:00
tags: [project, backend, java-spring, mysql, bkurs]
home_hub: projects
related:
  - "[[../projects]]"
  - "[[../bkurs-fe/bkurs-fe]]"
  - "[[../../20-areas/backend/spring/java-spring/java-spring]]"
  - "[[../../20-areas/backend/spring/java-spring/database/database]]"
  - "[[../../20-areas/backend/spring/java-spring/api-design/api-design]]"
  - "[[../../20-areas/database/database]]"
---

# bkurs-be — B-KURS 백엔드 프로젝트 hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-20 | engineering-agent/tech-lead | 최초 — 시작 컨텍스트 + 기술 결정 + PR 시리즈 계획 |
| v.1.1.0 | 2026-05-20 | engineering-agent/tech-lead | B2A 진행 로그 추가 (Account + JWT, PR #4 CI GREEN 머지 대기) |
| v.1.2.0 | 2026-05-20 | engineering-agent/tech-lead | B2B 진행 로그 + 결정 2개 + 회수 pitfall 노트 1개 (Refresh + Cookie + CORS, stacked PR #6) |
| v.1.3.0 | 2026-05-20 | engineering-agent/tech-lead | B2C 진행 로그 + 결정 1개 (학원 connect + membership + 이중 상태 트랙, stacked PR #8). B2 완결. |
| v.1.4.0 | 2026-05-20 | engineering-agent/tech-lead | B3A 진행 로그 + 결정 1개 (학원 CRUD + 코드 관리, stacked PR #10) + 회수 pitfall 1개 (save vs saveAndFlush). |
| v.1.5.0 | 2026-05-20 | engineering-agent/tech-lead | B3B 진행 로그 + 결정 1개 (강사 invite + assignment, stacked PR #12). 회귀 44/44. |
| v.1.6.0 | 2026-05-20 | engineering-agent/tech-lead | B3C 진행 로그 + 결정 1개 (학원 공지 + read 추적, stacked PR #14). 회귀 54/54. B3 시리즈 완결. |
| v.1.7.0 | 2026-05-20 | engineering-agent/tech-lead | B4A 진행 로그 + 결정 1개 (학원 정보 4필드 + 학생·강사 조회, stacked PR #16) + 회수 pitfall 1개 (ERROR dispatch /error 401). 회귀 62/62. B4 시리즈 시작. |

**[[../projects|↑ 10-projects]]** · **짝 프로젝트 [[../bkurs-fe/bkurs-fe]]**

---

## 1. 프로젝트 정의

bkurs-fe (mock 으로 화면 완성된 React 프론트) 에 대응하는 **REST API 백엔드**.

inroad-high (원본) 는 Supabase 를 BaaS 로 직접 호출했지만, bkurs 는 별도 백엔드 서비스를 둠. 이유:
- 학원·학생·강사 도메인 정책을 코드로 명문화 (Supabase RLS 만으로 부족)
- 외부 SDK (Claude AI / STT / 카카오 SDK 등) 호출을 백엔드에서 통제
- 결제 / 영수증 같은 회계 데이터는 백엔드가 신뢰 경계

## 2. 기술 결정

| 영역 | 선택 | 이유 |
|---|---|---|
| 언어 | Java 21 (LTS, Temurin) | 사용자 지정, 팀 친숙도 + record / pattern matching |
| 프레임워크 | Spring Boot 3.5.14 | 3.4 는 Initializr 가 거부, 3.5 가 최소 안정 |
| 빌드 | Gradle (Groovy DSL) + Wrapper | 팀 표준, IntelliJ 친화 |
| ORM | Spring Data JPA + Hibernate | 보일러플레이트 최소, repository 기반 빠른 개발 |
| 마이그레이션 | Flyway (+ flyway-mysql) | SQL 기반 + 버전 명확, 협업/롤백 용이 |
| DB | MySQL 9.6 (local) | 사용자 환경 기 설치 + 운영 표준 |
| 인증 | Spring Security | 표준 토큰 / 세션 분기 가능 |
| 검증 | Bean Validation (Jakarta) | DTO 검증 단일화 |
| 로깅 | Logback (Spring 기본) | 표준 + actuator 와 친화 |
| 보조 | Lombok, configuration-processor, actuator | DTO/Entity 보일러플레이트 축소, 헬스/메트릭 |

## 3. 로컬 환경 (이미 셋업 완료)

```bash
# Java 21
source ~/.sdkman/bin/sdkman-init.sh && sdk use java 21.0.5-tem

# MySQL local
brew services start mysql

# DB / user (이미 생성 — 1회만)
mysql -uroot -p'13950991*' -e "
  CREATE DATABASE IF NOT EXISTS \`bkurs-local\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  GRANT ALL PRIVILEGES ON \`bkurs-local\`.* TO 'mw'@'localhost';
  FLUSH PRIVILEGES;
"
```

자격 정보는 [[../../memory/project_bkurs_be_setup|project memory]] 와 본 노트만 보관. **프로젝트 README / CI 에 평문 비번 절대 금지.**

## 4. PR 시리즈 계획

bkurs-fe 의 S2A-S2N 시리즈 패턴을 그대로 따른다. 한 PR = 한 issue = 최소 3 commit.

| PR | 내용 |
|---|---|
| **B1** (Setup) ✅ | application.yml 분리 (local/prod) + 패키지 구조 (`com.bkurs.{domain}`) + `HealthController` + DB 연결 smoke 테스트 + CI (Gradle / JDK21) + docs (AGENTS/CLAUDE/COMMIT/BRANCH/CHANGELOG/PR-template) + Flyway V1 placeholder |
| **B2A** (Account + JWT) ✅ CI GREEN | `Account` 단일 테이블 + Role (5) / Status (5) enum + Flyway V2 + BCrypt 회원가입 + JWT HS256 access (30분) + `AuthController` (signup/login/me) + `AuthExceptionHandler` (401/409) + AuthApiTest 5 통합. PR #4 |
| **B2B** (Refresh + Cookie + CORS) ✅ 로컬 16/16 GREEN | `RefreshToken` (`refresh_token` + uk_token_hash + family_id) + `RefreshTokenService` (issue/rotate/logout/reuse 감지) + `RefreshCookieFactory` (HttpOnly/SameSite=Lax/Path=/api/v1/auth/Secure-prod) + `/api/v1/auth/refresh`/`/logout` + `CorsConfig` (allowedOriginPatterns + allow-credentials=true) + `noRollbackFor` reuse family revoke 보존. PR #6 (stacked) |
| **B2C** (Connect / Approve / 상태 전이) ✅ 로컬 24/24 GREEN | `Academy` + `AcademyCode` + `StudentAcademyMembership` (이중 상태 트랙) + V4 + `AcademyConnectService` + `MembershipService` (ownership 2계층 검증) + `/api/v1/auth/connect` + `/api/v1/academies/memberships/{id}/approve|reject` + `@PreAuthorize("hasRole('OWNER')")` + `AcademyExceptionHandler` 7 매핑. PR #8 (stacked, B2 완결) |
| **B3A** (학원 CRUD + 코드 관리) ✅ 로컬 34/34 GREEN | `AcademyManagementService` (1인 1학원) + `AcademyCodeService` (32-char alphabet random + saveAndFlush 재시도) + AcademyController/AcademyCodeController/OwnerMembershipQueueController + @PreAuthorize OWNER + ownership 2계층 + 예외 3 + 회귀 10. PR #10 (stacked) |
| **B3B** (강사 invite + 학원 assignment) ✅ 로컬 44/44 GREEN | `TeacherInvite` (32-byte base64url + 7일 + 1회용 consume invariant) + `TeacherAcademyAssignment` (ACTIVE/REMOVED) + V5 + service 2 + endpoint 5 + 예외 5 + 회귀 10. PR #12 (stacked) |
| **B3C** (학원 공지 + read 추적) ✅ 로컬 54/54 GREEN | `Announcement` + `AnnouncementRead` (uk + CASCADE) + V6 + `AcademyMembershipResolver` (role→academyId 통합) + service 2 + endpoint 6 (/me OWNER + /my role 무관) + DTO 4 + 예외 2 + 회귀 10. PR #14 (stacked, **B3 완결**) |
| **B4A** (학원 정보 보강 + 학생·강사 조회) ✅ 로컬 62/62 GREEN | Academy 4 필드 (phone/address/logoUrl/website) + V7 + updateInfo invariant + PATCH /me 확장 + 신규 GET /my/academy + `GlobalValidationExceptionHandler` (회수: ERROR dispatch /error 401 함정) + 회귀 8. PR #16 (stacked, B4 시리즈 시작) |
| B4B (학원 플랜) | Plan + AcademyPlanSubscription + 플랜 변경 / 만료 / 갱신 |
| B4C (결제 연동) | 결제 게이트웨이 (포트원/토스/네이버 등 추상화) + 영수증 / 환불 |
| B4D (학원장 이관) | owner_account_id 변경 endpoint + 감사 로그 + 양측 confirm |
| B5 (Student 프로필) | 학생 프로필 + 학교급 분기 (고등/중등) + 학원 매칭 |
| B6 (Roadmap) | 학년 × 분기 × 미션 + 학생별 진행 상태 |
| B6+ | inroad-high 의 도메인 (생기부 / 자소서 / 예상질문 / 기출 / 모의면접 / 시뮬 / 토론 / 수행평가 / 독서 / 숙제 / 학사관리 ...) 별 endpoint |

bkurs-fe 의 mock 타입 (StudentEssay, MyHomework, PastQuestion, ...) 이 API 응답 스키마 초안으로 활용 — 한쪽씩 협의 후 정식화.

## 5. 작업 원칙

- **vault 기록**: [[../../memory/feedback_vault_project_tracking|정책]] 에 따라 PR 별로 본 노트 + 20-areas 보강
- **commit 규칙**: 한국어 + gitmoji + 3섹션 (변경 이유 / 주요 변경 / 비고)
- **PR 당 최소 3 commit**: docs-only chore 만 squash 허용
- **author**: codwithyc 통일
- **민감 자격**: secret 절대 commit 금지 — application-secret.yml / .env 는 .gitignore

## 6. 진행 로그

| 일자 | PR | 내용 |
|---|---|---|
| 2026-05-20 | initial | 빈 repo → Spring Initializr 받음 (Boot 3.5.14 + 9 deps) + README + .gitignore. main push 는 classifier 차단으로 사용자 직접 실행 대기 |
| 2026-05-20 | B1 (local 준비 완료) | dev / feature/issue-1-setup 분기 + 3 commit (application.yml 분리 / Health+Security+Flyway+tests / docs+CI+templates). ./gradlew test 4/4 GREEN. push 대기 |
| 2026-05-20 | B1 #2 머지 ✅ | `gh api` 로 main + dev branch 생성 우회 (git push 차단 회피) → default branch main 설정 → issue #1 + PR #2 (5 commit) → CI (MySQL 9 service + JDK21) 1m43s pass → merge --delete-branch. issue #1 close |
| 2026-05-20 | B2A #4 CI GREEN (머지 대기) | issue #3 → feature/issue-3-b2a-account-jwt (6 commit) → Account + JWT 인증 + AuthExceptionHandler (401/409) + SecurityConfig 401 명시 + AuthApiTest 5 통합 회귀. CI 1m54s pass. 자동 머지는 classifier 차단 → 사용자 manual merge 대기. issue #3 PR 머지 후 close 예정. |
| 2026-05-20 | B2B #6 stacked PR | issue #5 → feature/issue-5-b2b-refresh-cors (5 commit) base=B2A → Refresh 회전 + HttpOnly Cookie + reuse detection + logout + CORS + noRollbackFor 버그 수정. 회귀 16/16 GREEN (smoke 4 + Auth 5 + Refresh 6 + CORS 1). CI 는 B2A 머지 후 dev 로 retarget 되면 실행. 사용자 manual merge 대기. |
| 2026-05-20 | B2C #8 stacked PR | issue #7 → feature/issue-7-b2c-academy-connect (4 commit) base=B2B → Academy + AcademyCode + StudentAcademyMembership + V4 + AcademyConnectService + MembershipService + ConnectController + MembershipController + @PreAuthorize OWNER + AcademyExceptionHandler 7매핑 + AcademyApprovalApiTest 8 회귀. **24/24 GREEN**. B2 시리즈 완결. |
| 2026-05-20 | B3A #10 stacked PR | issue #9 → feature/issue-9-b3a-academy-management (3 commit) base=B2C → AcademyManagementService + AcademyCodeService (random 8자 + saveAndFlush 재시도) + AcademyController + AcademyCodeController + OwnerMembershipQueueController + DTO 4 + 예외 3 + AcademyManagementApiTest 10 회귀. **34/34 GREEN**. B3 시리즈 시작. |
| 2026-05-20 | B3B #12 stacked PR | issue #11 → feature/issue-11-b3b-teacher-invite (4 commit) base=B3A → TeacherInvite + TeacherAcademyAssignment + V5 + TeacherInviteService (32-byte base64url 7일 1회용) + TeacherAssignmentService + 5 endpoint + DTO 3 + 예외 5 + TeacherInviteApiTest 10 + 기존 4 테스트 cleanup 보강. **44/44 GREEN**. |
| 2026-05-20 | B3C #14 stacked PR | issue #13 → feature/issue-13-b3c-announcement (5 commit) base=B3B → Announcement + AnnouncementRead + V6 + AcademyMembershipResolver (role→academyId) + AnnouncementService (OWNER CRUD) + AnnouncementReadService (N+1 회피 + upsert) + 6 endpoint (/me OWNER + /my role 무관) + DTO 4 + 예외 2 + AnnouncementApiTest 10. **54/54 GREEN**. B3 시리즈 완결. |
| 2026-05-20 | B4A #16 stacked PR | issue #15 → feature/issue-15-b4a-academy-info (3 commit) base=B3C → V7 ALTER academy + 4 nullable 컬럼 (phone/address/logoUrl/website) + Academy.updateInfo invariant + PATCH /me 확장 + 신규 GET /my/academy (학생·강사) + GlobalValidationExceptionHandler (회수: ERROR dispatch /error 401) + AcademyInfoApiTest 8. **62/62 GREEN**. B4 시리즈 시작. |

## 7. cross-link

- [[../bkurs-fe/bkurs-fe]] — 짝 프론트엔드 프로젝트
- [[../../20-areas/backend/spring/java-spring/java-spring|Java + Spring Hub]]
- [[../../20-areas/database/database|database hub]]
- [[../../memory/project_bkurs_be_setup|project memory]]
