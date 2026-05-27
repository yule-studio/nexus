---
title: "bkurs-be — 데이터베이스 / Flyway 마이그레이션"
kind: handover
status: stable
created_at: 2026-05-21T22:25:00+09:00
tags: [bkurs-be, handover, database, flyway, mysql]
related:
  - "[[../bkurs-be]]"
  - "[[catalog]]"
  - "[[overview]]"
home_hub: bkurs-be
---

# bkurs-be — 데이터베이스 / Flyway 마이그레이션

> "어느 컬럼 / FK / uk 가 어느 도메인에 속하나" + "새 migration 어떻게 쓰나". DB 자체는
> MySQL 9.6, ORM 은 Spring Data JPA (Hibernate), 마이그레이션은 **Flyway SQL based**.

## 1. 환경 / 연결

| 환경 | DB | 자격 |
| --- | --- | --- |
| local | `bkurs-local` (백틱 필수 — hyphen) on localhost:3306 | mw / `${DB_PASSWORD}` (env) |
| dev | 배포된 MySQL (dev compose) | env 주입 |
| prod | MySQL on EC2 + EBS gp3 data volume | SSM Parameter Store SecureString (DB_PASSWORD) |

application yml 키 (자세히 [[directory-structure#5]]):

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?...
    username: ${DB_USERNAME:mw}
    password: ${DB_PASSWORD:?DB_PASSWORD env required}
  jpa:
    hibernate:
      ddl-auto: validate     # Flyway 가 schema 변경, JPA 는 검증만
    open-in-view: false      # OSIV 비활성 — lazy 회피
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true
```

> `ddl-auto: validate` — entity vs schema mismatch 시 startup 실패. 새 컬럼 추가 시 entity +
> Flyway 둘 다 갱신해야 startup 성공.

## 2. 마이그레이션 카탈로그 V1~V29

| V | 파일 | 내용 | 도메인 | PR |
| --- | --- | --- | --- | --- |
| V1 | `V1__init_schema.sql` | placeholder | — | B1 |
| V2 | `V2__create_account.sql` | `account` (role 5 / status 5 / uk email) | auth | B2A |
| V3 | `V3__create_refresh_token.sql` | `refresh_token` (uk_token_hash + family_id) | auth | B2B |
| V4 | `V4__create_academy_and_membership.sql` | `academy` / `academy_code` (uk code) / `student_academy_membership` (이중 상태) | academy | B2C |
| V5 | `V5__create_teacher_invite_and_assignment.sql` | `teacher_invite` (uk token, 7일 1회용) / `teacher_academy_assignment` (ACTIVE/REMOVED) | academy | B3B |
| V6 | `V6__create_announcement_and_read.sql` | `announcement` / `announcement_read` (uk + CASCADE) | announcement | B3C |
| V7 | (academy 4 컬럼) | `ALTER academy ADD phone/address/logo_url/website` nullable | academy | B4A |
| V8 | `V8__...` | `plan` (catalog) / `academy_plan_subscription` (ACTIVE/CANCELLED/EXPIRED) | plan | B4B |
| V9 | `V9__...` | `payment` (PENDING/SUCCESS/FAILED/REFUNDED invariant) | plan | B4C |
| V10 | `V10__create_academy_owner_transfer.sql` | `academy_owner_transfer` (양측 confirm + actor invariant) | academy | B4D |
| V11 | `V11__create_student_profile.sql` | `student_profile` (account 1:1 + grade) | student | B5 |
| V12 | `V12__create_course_and_enrollment.sql` | `course` / `course_enrollment` (ACTIVE/DROPPED) | course | B6A |
| V13 | `V13__create_course_session_and_attendance.sql` | `course_session` (invariant end>start) / `attendance` (4 status + uk upsert + recorded_by) | course | B6B |
| V14 | `V14__create_homework_and_submission.sql` | `homework` / `homework_submission` (uk + grade audit) | homework | B7A |
| V15 | `V15__create_personal_statement_and_feedback.sql` | `personal_statement` (DRAFT/SUBMITTED) / `statement_feedback` | statement | B7B |
| V16 | `V16__create_question_and_attempt.sql` | `question` (type EXPECTED/PAST) / `question_attempt` (uk + score) | question | B7C |
| V17 | `V17__create_mock_interview_and_response.sql` | `mock_interview` (type 4) / `interview_response` (uk + audio/video URL + 평가) | interview | B7D |
| V18 | `V18__create_student_record_entry.sql` | `student_record_entry` (8 카테고리) | record | B7E |
| V19 | `V19__create_discussion.sql` | `discussion_topic` / `discussion_post` (uk topic+account + FOR/AGAINST/NEUTRAL) | discussion | B7F |
| V20 | `V20__create_reading_note.sql` | `reading_note` / `reading_feedback` (uk note+teacher + recommended_score 0~5) | reading | B7G |
| V21 | `V21__create_performance_evaluation.sql` | `performance_evaluation` (max_score 1-1000) / `performance_submission` (uk + resubmit 채점 무효화) | performance | B7H |
| V22 | `V22__create_simulation.sql` | `simulation_case` / `simulation_response` (uk + resubmit review 무효화 + feasibility 1-5) | simulation | B7I |
| V23 | `V23__alter_student_record_entry_source.sql` | `ALTER student_record_entry ADD source_type/source_id + uk` | record | B9 |
| V24 | `V24__create_spring_batch_tables.sql` | Spring Batch metadata (`BATCH_JOB_INSTANCE` …) | batch | (Phase A) |
| V25 | `V25__create_concept_diagnosis.sql` | `concept_diagnosis` | concept | (사이클 2) |
| V26 | `V26__create_suhaeng.sql` | `suhaeng_round` 등 | suhaeng | (사이클 2) |
| V27 | `V27__create_middle_saenggibu.sql` | 중등 생기부 (semester lock) | saenggibu | (사이클 2) |
| V28 | `V28__create_past_question.sql` | `past_question` (master 발행) | admission | PR #152 |
| V29 | `V29__create_major_question.sql` | `major_question` (master 발행) | admission | PR #153 |

> grade 범위 확장 (StudentProfile.grade 1~3 → 1~6) 은 V 신규 없이 **entity + DTO 만 수정** (PR
> #150). 컬럼 type 자체는 V11 의 INT 라 호환.

## 3. 명명 규칙

| 항목 | 규칙 | 예 |
| --- | --- | --- |
| 테이블 | snake_case 단수 (Spring Data 관습) | `student_record_entry` |
| 컬럼 | snake_case | `recorded_by_account_id`, `created_at` |
| PK | `id` BIGINT AUTO_INCREMENT | |
| FK | `{relation}_id` | `account_id`, `course_id`, `academy_id` |
| audit | `created_at`, `updated_at`, `created_by`, `updated_by` | TIMESTAMP DEFAULT NOW |
| status | ENUM (Java) + VARCHAR(32) (DB) | `ACTIVE` / `DROPPED` |
| unique | `uk_{table}_{cols}` | `uk_refresh_token_token_hash` |
| index | `idx_{table}_{cols}` | `idx_homework_course_id` |
| FK constraint | `fk_{table}_{ref}` | `fk_attendance_session_id` |

charset / collation: utf8mb4 / utf8mb4_unicode_ci (한글 + emoji 안전).

## 4. invariant 위치

| invariant | 어디서 강제 |
| --- | --- |
| `success ↔ data` envelope | `ApiEnvelope` record 생성자 |
| `Academy 1 OWNER 1 학원` | `AcademyManagementService.createForOwner` + DB uk_academy_owner_account_id |
| `Course.end > start` | `Course` setter / migration CHECK (MySQL 8+ check) |
| `Refresh family revoke` | `RefreshTokenService.rotate` + `@Transactional(noRollbackFor = ...)` |
| `record source unique upsert` | V23 `uk(source_type, source_id)` |
| `Submission uk` | V14 `uk(homework_id, student_account_id)` — upsert 강제 |
| `Performance resubmit 채점 무효화` | V21 + `PerformanceSubmission.resubmit()` |
| `Simulation resubmit review 무효화` | V22 + `SimulationResponse.resubmit()` |

DB 단의 uk / CHECK 와 entity method 양쪽에 둠 — 한쪽만 두면 우회 가능. 신규 도메인 시 둘 다.

## 5. 새 마이그레이션 작성 절차

1. **다음 번호** 확인 — 현재 V29. 다음은 V30.
2. 파일: `src/main/resources/db/migration/V30__create_{snake_case}.sql`
3. 내용 작성 (DDL 만, 데이터 변경은 별도 V).
4. entity (`{domain}/domain/{Entity}.java`) — `@Entity`, `@Table(name="...")` 일치, 컬럼 매핑.
5. Repository (`{Entity}Repository extends JpaRepository<...>`).
6. `./gradlew clean check` — Flyway 가 자동 apply + JPA validate.
7. 통합 테스트 (`AbstractIntegrationTest extends`) 회귀 추가 — [[testing-guide]].

> ⚠️ **이미 prod 에 apply 된 V 파일 수정 금지** — Flyway checksum mismatch 로 startup fail.
> 수정이 필요하면 다음 V 로 ALTER.

## 6. local DB 준비 (1회)

```bash
brew services start mysql

mysql -uroot -p -e "
  CREATE DATABASE IF NOT EXISTS \`bkurs-local\`
    DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  CREATE USER IF NOT EXISTS 'mw'@'localhost' IDENTIFIED BY '\$DB_PASSWORD';
  GRANT ALL PRIVILEGES ON \`bkurs-local\`.* TO 'mw'@'localhost';
  FLUSH PRIVILEGES;
"
```

또는 docker-compose:

```bash
docker compose -f docker-compose.dev.yml --env-file .env.local up -d mysql
```

## 7. 운영 시 주의

- `validate-on-migrate=true` — checksum / 누락 시 startup fail. 의도된 보호.
- `baseline-on-migrate=false` (prod) — baseline 자동화 회피. 신규 환경은 빈 DB 부터.
- 큰 ALTER (예: index 추가 on 100M row) 는 Flyway 가 아닌 별도 online schema change 도구 검토.

## 8. test DB cleanup

`AbstractIntegrationTest` 가 `@AfterEach` 로 모든 테이블 truncate (관계 역순). FK 충돌 회피
순서 유지. 새 테이블 추가 시 cleanup 순서에 **반드시 추가** — 자세히 [[testing-guide#cleanup]].

## cross-link

- [[../bkurs-be]]
- [[catalog]] — 도메인별 V 번호 1:1 매핑
- [[overview]] — JPA / 트랜잭션 / OSIV
- [[testing-guide]] — Abstract test cleanup 순서
