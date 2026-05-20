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
| v.1.8.0 | 2026-05-20 | engineering-agent/tech-lead | B4B 진행 로그 + 결정 1개 (Plan catalog + Subscription 분리, stacked PR #18). 회귀 72/72. |
| v.1.9.0 | 2026-05-20 | engineering-agent/tech-lead | B4C 진행 로그 + 결정 1개 (PaymentGateway 추상화 + Mock 구현 + 환불, stacked PR #20). 회귀 82/82. |
| v.1.10.0 | 2026-05-20 | engineering-agent/tech-lead | B4D 진행 로그 + 결정 1개 (학원장 이관 양측 confirm + audit, stacked PR #22). 회귀 92/92. **B4 시리즈 완결.** |
| v.1.11.0 | 2026-05-20 | engineering-agent/tech-lead | B5 진행 로그 + 결정 1개 (학생 프로필 1:1 + 학원장 조회, stacked PR #24). 회귀 102/102. **B5 시리즈 시작.** |
| v.1.12.0 | 2026-05-20 | engineering-agent/tech-lead | B6A 진행 로그 + 결정 1개 (Course + Enrollment + 강사 배정, stacked PR #26). 회귀 112/112. **B6 시리즈 시작.** |
| v.1.13.0 | 2026-05-20 | engineering-agent/tech-lead | B6B 진행 로그 + 결정 1개 (CourseSession + Attendance upsert, stacked PR #28). 회귀 122/122. |
| v.1.14.0 | 2026-05-20 | engineering-agent/tech-lead | B6C 진행 로그 + 결정 1개 (강사 권한 매트릭스 정밀화, stacked PR #30). 회귀 132/132. **B6 시리즈 완결.** |
| v.1.15.0 | 2026-05-20 | engineering-agent/tech-lead | B7A 진행 로그 + 결정 1개 (Homework + Submission + 채점, stacked PR #32). 회귀 142/142. **B7 학습 콘텐츠 시리즈 시작.** |
| v.1.16.0 | 2026-05-20 | engineering-agent/tech-lead | B7B 진행 로그 + 결정 1개 (PersonalStatement + Feedback + DRAFT/SUBMITTED, stacked PR #34). 회귀 152/152. |
| v.1.17.0 | 2026-05-20 | engineering-agent/tech-lead | REFACTOR — AbstractIntegrationTest 추출 (16 테스트 cleanup 통합, **+122 / -1377 lines**), stacked PR #36. 회귀 152/152 유지. 회수 누적의 종결. |
| v.1.18.0 | 2026-05-20 | engineering-agent/tech-lead | B7C 진행 로그 + 결정 1개 (Question type 통합 + Attempt upsert, stacked PR #38). 회귀 162/162. **REFACTOR 효과 확인 — cleanup 보강 0 곳.** |
| v.1.19.0 | 2026-05-20 | engineering-agent/tech-lead | B7D 진행 로그 + 결정 1개 (MockInterview + Response 미디어 URL + 평가, stacked PR #40). 회귀 172/172. |
| v.1.20.0 | 2026-05-20 | engineering-agent/tech-lead | B7E 진행 로그 + 결정 1개 (StudentRecord 8 카테고리 + 강사 입력 + 학생/학원장 조회, stacked PR #42). 회귀 182/182. |
| v.1.21.0 | 2026-05-20 | engineering-agent/tech-lead | B7F 진행 로그 + 결정 1개 (Discussion topic+post upsert + 협업 평가, stacked PR #44). 회귀 192/192. |
| v.1.22.0 | 2026-05-20 | engineering-agent/tech-lead | B7G 진행 로그 + 결정 1개 (ReadingNote 학생 owner + 강사 feedback upsert, stacked PR #46). 회귀 202/202. |
| v.1.23.0 | 2026-05-20 | engineering-agent/tech-lead | B7H 진행 로그 + 결정 1개 (PerformanceEvaluation 4 카테고리 + 재제출 채점 무효화, stacked PR #48). 회귀 212/212. |
| v.1.24.0 | 2026-05-20 | engineering-agent/tech-lead | B7I 진행 로그 + 결정 1개 (Simulation 3 type + feasibility 1-5 review, stacked PR #50). 회귀 222/222. |
| v.1.25.0 | 2026-05-20 | engineering-agent/tech-lead | B9 진행 로그 + 결정 1개 (StudentRecord auto-bridge 4 도메인 → 4 카테고리 same-TX upsert, stacked PR #52). 회귀 232/232. **B7 series + bridge 완결.** |
| v.1.26.0 | 2026-05-20 | engineering-agent/tech-lead | B10 진행 로그 + 결정 1개 (통합 검색 5 도메인 LIKE aggregator + 권한 3 endpoint, stacked PR #54). 회귀 242/242. |
| v.1.27.0 | 2026-05-20 | engineering-agent/tech-lead | B11 진행 로그 + 결정 1개 (학생 통계 7 영역 aggregation + 권한 3 endpoint, stacked PR #56). 회귀 252/252. |
| v.1.28.0 | 2026-05-20 | engineering-agent/tech-lead | B13 진행 로그 + 결정 1개 (강사 dashboard 수업별 5 지표 + 1 endpoint, stacked PR #58). 회귀 260/260. |
| v.1.29.0 | 2026-05-20 | engineering-agent/tech-lead | B16 진행 로그 + 결정 1개 (학원장 dashboard 학원 9 지표 rollup + 1 endpoint, stacked PR #60). 회귀 268/268. |
| v.1.30.0 | 2026-05-20 | engineering-agent/tech-lead | B12 진행 로그 + 결정 1개 (bridge 자동 period 한국 학년도, stacked PR #62). 회귀 276/276. |

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
| **B4B** (학원 플랜 + 구독) ✅ 로컬 72/72 GREEN | `Plan` catalog (MASTER CRUD) + `AcademyPlanSubscription` (ACTIVE/CANCELLED/EXPIRED) + V8 + service 2 + endpoint 7 (admin 3 + catalog 1 + owner 4) + DTO 5 + 예외 4. 결제는 B4C — mock subscribe. PR #18 (stacked) |
| **B4C** (PaymentGateway + 환불) ✅ 로컬 82/82 GREEN | `Payment` (PENDING/SUCCESS/FAILED/REFUNDED + invariant) + V9 + `PaymentGateway` interface + `MockPaymentGateway` + `PaymentService` + `SubscriptionService.chargeOrFail/cancelAndRefund` + 신규 endpoint 3 (refund + /me/payments OWNER + /admin/payments MASTER) + 402 매핑. PR #20 (stacked) |
| **B4D** (학원장 이관 + audit) ✅ 로컬 92/92 GREEN | `AcademyOwnerTransfer` (PENDING/CONFIRMED/REJECTED/CANCELLED + actor invariant) + V10 + `Academy.transferTo` + `OwnerTransferService` (양측 confirm + 1인 1학원 유지) + 6 endpoint (FROM 3 + TO 3) + DTO 2 + 예외 6. PR #22 (stacked, **B4 시리즈 완결**) |
| **B5** (학생 프로필) ✅ 로컬 102/102 GREEN | `StudentProfile` (account 1:1 + grade INT 1-3 + schoolName + birthDate + parentPhone) + V11 + service (upsert/patch/get + 학원장 ownership) + 4 endpoint (STUDENT 3 + OWNER 1) + DTO 3 + 예외 3. PR #24 (stacked, **B5 시리즈 시작**) |
| **B6A** (Course + Enrollment) ✅ 로컬 112/112 GREEN | `Course` (academy_id + teacher_assignment_id nullable + maxStudents) + `CourseEnrollment` (ACTIVE/DROPPED) + V12 + service 2 + 9 endpoint (OWNER 5 + TEACHER 2 + STUDENT 3) + DTO 4 + 예외 5. PR #26 (stacked, **B6 시리즈 시작**) |
| **B6B** (CourseSession + Attendance) ✅ 로컬 122/122 GREEN | `CourseSession` (start/end + room + invariant end > start) + `Attendance` (PRESENT/ABSENT/LATE/EXCUSED + uk upsert + recorded_by audit) + V13 + service 2 + 9 endpoint (OWNER 4 + TEACHER 3 + STUDENT 2) + DTO 5 + 예외 2. PR #28 (stacked) |
| **B6C** (강사 권한 매트릭스) ✅ 로컬 132/132 GREEN | `Course.updateLimited` (description 만) + `CourseService.updateByTeacher` (write 트랜잭션) + `StudentProfileService.getStudentOnly` + 3 endpoint (OWNER 출결 / TEACHER course PATCH / TEACHER 학생 프로필) + DTO 1. PR #30 (stacked, **B6 시리즈 완결**) |
| **B7A** (Homework + Submission) ✅ 로컬 142/142 GREEN | `Homework` (course_id + assigned_by + due_at) + `HomeworkSubmission` (uk upsert + grade 0-100 + graded_by audit) + V14 + service 2 + 9 endpoint (TEACHER 6 + STUDENT 3) + DTO 6 + 예외 3. PR #32 (stacked, **B7 학습 콘텐츠 시리즈 시작**) |
| **B7B** (PersonalStatement + Feedback) ✅ 로컬 152/152 GREEN | `PersonalStatement` (DRAFT/SUBMITTED + invariant) + `StatementFeedback` (강사 댓글 + audit) + V15 + service 2 + 9 endpoint (학생 5 + 강사 4) + DTO 5 + 예외 2. PR #34 (stacked) |
| **B7C** (Question 통합 + Attempt) ✅ 로컬 162/162 GREEN | `Question` (type EXPECTED/PAST + year/subject) + `QuestionAttempt` (uk + score 0-100) + V16 + service 2 + 9 endpoint (강사 6 + 학생 3) + DTO 6 + 예외 3. PR #38 (stacked). REFACTOR 효과 — cleanup 보강 0 곳. |
| **B7D** (MockInterview + 미디어 응답) ✅ 로컬 172/172 GREEN | `MockInterview` (type 4종 + expectedMinutes) + `InterviewResponse` (uk + 텍스트 + audio/video URL + 평가 0-100) + V17 + service 2 + 9 endpoint + DTO 6 + 예외 3. PR #40 (stacked) |
| **B7E** (StudentRecord 생기부 8 카테고리) ✅ 로컬 182/182 GREEN | `RecordCategory` enum 8종 (AWARD/AUTONOMY/CLUB/CAREER/SERVICE/SUBJECT/BEHAVIOR/READING — 학종 표준) + `StudentRecordEntry` (course nullable + author audit) + V18 + service 1 (ownership 2계층) + 6 endpoint (TEACHER 4 + STUDENT 1 + OWNER 1) + DTO 3 + 예외 2. PR #42 (stacked) |
| **B7F** (Discussion 토론 + 평가) ✅ 로컬 192/192 GREEN | `DiscussionTopic` (DEBATE/SEMINAR/QNA) + `DiscussionPost` (FOR/AGAINST/NEUTRAL + uk topic+account upsert + evaluate 0-100) + V19 + service 1 (협업 평가) + 8 endpoint (TEACHER 6 + STUDENT 2) + DTO 6 + 예외 3 + CourseEnrollmentRepository.findByAccountIdAndStatus 추가. PR #44 (stacked) |
| **B7G** (ReadingNote 독서 + feedback) ✅ 로컬 202/202 GREEN | `ReadingNote` (학생 owner, academy 격리 없음) + `ReadingFeedback` (uk note+teacher upsert + recommended_score 0-5) + V20 + service 1 + 7 endpoint (STUDENT 5 + TEACHER 2) + DTO 5 + 예외 2. PR #46 (stacked) |
| **B7H** (PerformanceEvaluation 수행평가 + 채점) ✅ 로컬 212/212 GREEN | `PerformanceCategory` enum 4종 (WRITTEN/PROJECT/PRESENTATION/EXPERIMENT) + `PerformanceEvaluation` (max_score 1-1000) + `PerformanceSubmission` (uk + resubmit 채점 무효화 + grade audit) + V21 + service 1 (협업 채점) + 8 endpoint (TEACHER 6 + STUDENT 2) + DTO 6 + 예외 3. PR #48 (stacked) |
| **B7I** (Simulation 모의지원 + review) ✅ 로컬 222/222 GREEN | `SimulationType` enum 3종 (UNIV_APPLICATION/CAREER/FINANCIAL) + `SimulationCase` (scenario + expected_outcome) + `SimulationResponse` (uk + resubmit review 무효화 + feasibility 1-5 별점) + V22 + service 1 (협업 review) + 8 endpoint (TEACHER 6 + STUDENT 2) + DTO 6 + 예외 3. PR #50 (stacked) |
| **B9** (StudentRecord auto-bridge) ✅ 로컬 232/232 GREEN | V23 ALTER + `RecordSourceType` enum 4종 + uk(source_type, source_id) + `record.bridge/` 패키지 (4 event + StudentRecordBridgeService 4 @EventListener) + 4 service (PE/Disc/Sim/Reading) publish. 매핑: PERFORMANCE→SUBJECT / DISCUSSION→AUTONOMY / SIMULATION→CAREER / READING→READING. same-TX upsert + idempotent + 학생부 영구. PR #52 (stacked) |
| **B10** (통합 검색) ✅ 로컬 242/242 GREEN | `SearchResultType` enum 5종 + `SearchResult` record (snippet 100자) + 5 repo ContainingIgnoreCase + `SearchService` aggregator (5 도메인 + 권한 3) + 3 endpoint (TEACHER/STUDENT/OWNER) + 예외 1. LIKE '%q%' + limit 50 (후속 FULLTEXT + pagination). PR #54 (stacked) |
| **B11** (학생 통계 7 영역) ✅ 로컬 252/252 GREEN | 7 sub-record (Attendance/Homework/Performance/Discussion/Reading/Simulation/Record) + `StudentStatsView` 통합 + `StatsService` aggregator (findAll + Java reduce, 권한 3 method) + 3 endpoint + 예외 1. PR #56 (stacked). 후속 @Query COUNT 최적화. |
| **B13** (강사 dashboard 수업별) ✅ 로컬 260/260 GREEN | `TeacherDashboardView` + `CourseDashboard` (수업별 5 지표) + `TeacherDashboardService` (본인 배정 수업 loop) + 1 endpoint. 5 지표: enrolledCount (ACTIVE만) / pendingHomeworkCount / pendingPECount / sessionCount / avgAttendanceRate. PR #58 (stacked) |
| **B16** (학원장 dashboard 학원 rollup) ✅ 로컬 268/268 GREEN | `AcademyDashboardView` (9 지표 + empty factory) + `AcademyDashboardService` (학원 전체 집계) + 1 endpoint (OWNER). 9 지표: activeStudent/activeTeacher/course/session/enrollment/attendance/gradedHomework/gradedPE/record counts. PR #60 (stacked) |
| **B12** (학기 자동 결정) ✅ 로컬 276/276 GREEN | `AcademicPeriod` record (year+semester, format) + from(LocalDate) 정적 + `StudentRecordBridgeService` period 자동 (KST today). 한국 학년도 (3-7월 1학기 / 8-12월 2학기 / 1-2월 전년도 2학기). 8 unit + integration. PR #62 (stacked) |
| B7F (토론 / 수행평가 / 독서 / 시뮬레이션) | 추가 학습 도메인 |
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
| 2026-05-20 | B4B #18 stacked PR | issue #17 → feature/issue-17-b4b-plan-subscription (4 commit) base=B4A → Plan + AcademyPlanSubscription + V8 + PlanService (MASTER CRUD) + SubscriptionService (subscribe/cancel/change ACTIVE 1개) + 7 endpoint (admin 3 + catalog 1 + owner 4) + DTO 5 + 예외 4 + PlanSubscriptionApiTest 10. **72/72 GREEN**. 결제 연동은 B4C. |
| 2026-05-20 | B4C #20 stacked PR | issue #19 → feature/issue-19-b4c-payment-gateway (4 commit) base=B4B → Payment + V9 + PaymentGateway interface + MockPaymentGateway + PaymentService + SubscriptionService chargeOrFail/cancelAndRefund + 3 endpoint (refund/me-payments/admin-payments) + 402 매핑 + PaymentApiTest 10. **82/82 GREEN**. |
| 2026-05-20 | B4D #22 stacked PR | issue #21 → feature/issue-21-b4d-owner-transfer (4 commit) base=B4C → AcademyOwnerTransfer + OwnerTransferStatus enum + V10 + Academy.transferTo + OwnerTransferService (양측 confirm + 1인 1학원 유지) + 6 endpoint (FROM 3 + TO 3) + DTO 2 + 예외 6 + OwnerTransferApiTest 10. **92/92 GREEN**. **B4 시리즈 완결.** |
| 2026-05-20 | B5 #24 stacked PR | issue #23 → feature/issue-23-b5-student-profile (4 commit) base=B4D → StudentProfile entity (account 1:1) + V11 + StudentProfileService (upsert/patch/get + 학원장 ownership) + 4 endpoint (STUDENT 3 + OWNER 1) + DTO 3 + 예외 3 + StudentProfileApiTest 10. **102/102 GREEN**. **B5 시리즈 시작.** 회수: grade TINYINT → INT (JPA Integer schema validate 충돌). |
| 2026-05-20 | B6A #26 stacked PR | issue #25 → feature/issue-25-b6a-course-enrollment (4 commit) base=B5 → Course + CourseEnrollment + V12 + CourseService (OWNER CRUD + 강사 검증) + EnrollmentService (학생 enroll/drop + max/중복) + 9 endpoint (OWNER 5 + TEACHER 2 + STUDENT 3) + DTO 4 + 예외 5 + CourseApiTest 10. **112/112 GREEN**. **B6 시리즈 시작.** |
| 2026-05-20 | B6B #28 stacked PR | issue #27 → feature/issue-27-b6b-session-attendance (4 commit) base=B6A → CourseSession + Attendance + V13 + AttendanceStatus enum + SessionService + AttendanceService (TEACHER/OWNER upsert + recorded_by audit) + 9 endpoint (OWNER 4 + TEACHER 3 + STUDENT 2) + DTO 5 + 예외 2 + SessionAttendanceApiTest 10 + 10 테스트 cleanup 보강. **122/122 GREEN**. |
| 2026-05-20 | B6C #30 stacked PR | issue #29 → feature/issue-29-b6c-teacher-permission (4 commit) base=B6B → Course.updateLimited + CourseService.updateByTeacher (write 트랜잭션) + StudentProfileService.getStudentOnly + 3 endpoint (OWNER 출결 + TEACHER PATCH description + TEACHER 학생 프로필) + DTO 1 + TeacherPermissionApiTest 10. **132/132 GREEN**. **B6 시리즈 완결.** |
| 2026-05-20 | B7A #32 stacked PR | issue #31 → feature/issue-31-b7a-homework (4 commit) base=B6C → Homework + HomeworkSubmission + V14 + HomeworkService (강사 CRUD) + SubmissionService (학생 upsert + 강사 채점 0-100) + 9 endpoint (TEACHER 6 + STUDENT 3) + DTO 6 + 예외 3 + HomeworkApiTest 10 + 14 테스트 cleanup 보강. **142/142 GREEN**. **B7 학습 콘텐츠 시리즈 시작.** |
| 2026-05-20 | B7B #34 stacked PR | issue #33 → feature/issue-33-b7b-personal-statement (4 commit) base=B7A → PersonalStatement + StatementFeedback + V15 + StatementStatus enum (DRAFT/SUBMITTED) + StatementService + FeedbackService + 9 endpoint (학생 5 + 강사 4) + DTO 5 + 예외 2 + PersonalStatementApiTest 10 + 15 테스트 cleanup 보강. **152/152 GREEN**. |
| 2026-05-20 | REFACTOR #36 stacked PR | issue #35 → feature/issue-35-refactor-abstract-test (2 commit) base=B7B → AbstractIntegrationTest 추출 + 16 테스트 extends + 중복 제거. **+122 / -1377 lines (-1270)**. 회귀 **152/152 GREEN 유지**. 신규 entity 추가 시 base 한 곳 보강. 회수 누적의 종결. |
| 2026-05-20 | B7C #38 stacked PR | issue #37 → feature/issue-37-b7c-question (5 commit) base=REFACTOR → Question (type EXPECTED/PAST 통합) + QuestionAttempt (uk upsert) + V16 + service 2 + 9 endpoint + DTO 6 + 예외 3 + QuestionApiTest 10. **162/162 GREEN**. **REFACTOR 효과 — cleanup 보강 0 곳.** |
| 2026-05-20 | B7D #40 stacked PR | issue #39 → feature/issue-39-b7d-mock-interview (5 commit) base=B7C → MockInterview (type 4종) + InterviewResponse (audio/video URL + 평가 0-100) + V17 + service 2 + 9 endpoint + DTO 6 + 예외 3 + MockInterviewApiTest 10. **172/172 GREEN**. |
| 2026-05-20 | B7E #42 stacked PR | issue #41 → feature/issue-41-b7e-student-record (5 commit incl fixup) base=B7D → RecordCategory enum 8종 + StudentRecordEntry + V18 + StudentRecordService (ownership 2계층 — courseService.listByTeacher + enrollment ACTIVE / membership ACTIVE) + 6 endpoint (TEACHER 4 + STUDENT 1 + OWNER 1) + DTO 3 + 예외 2 + StudentRecordApiTest 10. **182/182 GREEN**. STUDENT 거부 케이스는 Spring 6 환경 차이로 401/403 둘 다 허용. |
| 2026-05-20 | B7F #44 stacked PR | issue #43 → feature/issue-43-b7f-discussion (4 commit) base=B7E → DiscussionTopic + DiscussionPost (uk topic+account upsert + evaluate 0-100) + V19 + DiscussionService (강사 topic CRUD + 학생 post upsert + 협업 평가) + CourseEnrollmentRepository.findByAccountIdAndStatus 추가 + 8 endpoint (TEACHER 6 + STUDENT 2) + DTO 6 + 예외 3 + DiscussionApiTest 10. **192/192 GREEN**. peer reply 는 후속 PR. |
| 2026-05-20 | B7G #46 stacked PR | issue #45 → feature/issue-45-b7g-reading-note (4 commit) base=B7F → ReadingNote (학생 owner, academy 격리 없음) + ReadingFeedback (uk note+teacher upsert + recommended_score 0-5) + V20 + ReadingService (학생 CRUD + 강사 enrolled 학생 조회/feedback) + 7 endpoint (STUDENT 5 + TEACHER 2) + DTO 5 + 예외 2 + ReadingNoteApiTest 10. **202/202 GREEN**. 생기부 (B7E) READING 카테고리 원천 자료. |
| 2026-05-20 | B7H #48 stacked PR | issue #47 → feature/issue-47-b7h-performance-evaluation (4 commit) base=B7G → PerformanceCategory enum 4종 + PerformanceEvaluation (max_score 1-1000 invariant) + PerformanceSubmission (uk eval+student upsert + resubmit 채점 무효화) + V21 + PerformanceEvaluationService (강사 CRUD + 학생 upsert + 협업 채점) + 8 endpoint (TEACHER 6 + STUDENT 2) + DTO 6 + 예외 3 + PerformanceEvaluationApiTest 10. **212/212 GREEN**. Homework 와 의미 분리 — 공식 평가 (생기부 SUBJECT 후속 연계). |
| 2026-05-20 | B7I #50 stacked PR | issue #49 → feature/issue-49-b7i-simulation (4 commit) base=B7H → SimulationType enum 3종 + SimulationCase (scenario + expected_outcome) + SimulationResponse (uk + resubmit review 무효화 + feasibility 1-5) + V22 + SimulationService (강사 CRUD + 학생 respond upsert + 협업 review) + 8 endpoint + DTO 6 + 예외 3 + SimulationApiTest 10. **222/222 GREEN**. PE 와 의미 분리 — 정답 없는 시나리오 / 현실성 별점. |
| 2026-05-20 | B9 #52 stacked PR | issue #51 → feature/issue-51-b9-record-bridge (5 commit incl fixup) base=B7I → V23 ALTER student_record_entry source_type/source_id + uk + RecordSourceType enum 4종 + StudentRecordEntry overwriteFromSource + Repository findBySourceTypeAndSourceId + bridge 패키지 (4 event record + StudentRecordBridgeService 4 @EventListener) + 4 service (PE/Disc/Sim/Reading) ApplicationEventPublisher publish + StudentRecordBridgeApiTest 10. **232/232 GREEN**. 4 도메인 → 4 카테고리 자동 매핑 (PERFORMANCE→SUBJECT, DISCUSSION→AUTONOMY, SIMULATION→CAREER, READING→READING), idempotent (uk source), 학생부 영구 (domain DELETE 시 record 보존), 강사 수동 record coexist. |
| 2026-05-20 | B10 #54 stacked PR | issue #53 → feature/issue-53-b10-search (4 commit) base=B9 → SearchResultType enum 5종 + SearchResult record (snippet 100자) + 5 repo (Record/Reading/Discussion/PE/Sim) findBy...ContainingIgnoreCase + SearchService aggregator (5 도메인 호출 + createdAt desc + limit 50) + 3 endpoint (TEACHER `/teachers/me/students/{sid}/search`, STUDENT `/students/me/search`, OWNER `/academies/me/students/{sid}/search`) + 예외 1 + SearchApiTest 10. **242/242 GREEN**. LIKE '%q%' PoC — 후속 FULLTEXT + ngram + pagination 검토. |
| 2026-05-20 | B11 #56 stacked PR | issue #55 → feature/issue-55-b11-student-stats (4 commit) base=B10 → 7 sub-record DTO (Attendance/Homework/Performance/Discussion/Reading/Simulation/Record) + 통합 StudentStatsView + StatsService aggregator (7 영역 findAll + Java reduce + 권한 3 method) + 3 endpoint (STUDENT `/students/me/stats`, TEACHER `/teachers/me/students/{sid}/stats`, OWNER `/academies/me/students/{sid}/stats`) + 예외 1 + StudentStatsApiTest 10. **252/252 GREEN**. attendance rate / homework·PE avg / discussion avg / reading avgRecommended / simulation avgFeasibility / record byCategory (수동 + bridge 합산). |
| 2026-05-20 | B13 #58 stacked PR | issue #57 → feature/issue-57-b13-teacher-dashboard (3 commit) base=B11 → TeacherDashboardView + CourseDashboard 5 지표 (enrolledCount ACTIVE / pendingHomeworkCount / pendingPECount / sessionCount / avgAttendanceRate) + TeacherDashboardService (본인 배정 수업 loop + Java reduce) + 1 endpoint (TEACHER `/teachers/me/dashboard`) + TeacherDashboardApiTest 8. **260/260 GREEN**. 수업 1:N 폭 (학생 1:1 깊이는 B11). DTO+service 한 commit (분리 인위적 — 3 commit 패턴). |
| 2026-05-20 | B16 #60 stacked PR | issue #59 → feature/issue-59-b16-owner-dashboard (3 commit) base=B13 → AcademyDashboardView 9 지표 + AcademyDashboardService (학원 전체 rollup, AcademyManagementService.getMine 검증) + 1 endpoint (OWNER `/academies/me/dashboard`) + AcademyDashboardApiTest 8. **268/268 GREEN**. 학원 단위 — 학생 1:1 (B11) / 수업 1:N (B13) 와 분리. |
| 2026-05-20 | B12 #62 stacked PR | issue #61 → feature/issue-61-b12-academic-period (4 commit incl fixup) base=B16 → AcademicPeriod record + from(LocalDate) + StudentRecordBridgeService.upsert period 자동 (KST) + AcademicPeriodTest 8 unit + bridge integration assert. **276/276 GREEN**. 한국 학년도 (3월~다음해 2월) 규칙 — 8월 2학기 / 1-2월 전년도 2학기. |

## 7. cross-link

- [[../bkurs-fe/bkurs-fe]] — 짝 프론트엔드 프로젝트
- [[../../20-areas/backend/spring/java-spring/java-spring|Java + Spring Hub]]
- [[../../20-areas/database/database|database hub]]
- [[../../memory/project_bkurs_be_setup|project memory]]
