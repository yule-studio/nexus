---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, be, domain, architecture, reading]
related:
  - "[[00-index]]"
  - "[[01-project-map]]"
  - "[[03-be-api-contract]]"
---

# 04 · 도메인 아키텍처 — 읽는 법

> 본 문서는 **현재 BE 도메인 아키텍처를 어떻게 읽어야 하는가** 의 가이드. "지금 당장 고치기" 가 아니라 "지금 이대로 읽고 운영할 때 어떤 mental model 이 필요한가" 에 초점.
>
> Source: `src/main/java/com/bkurs/*` 도메인 패키지 구조 + controller prefix 패턴 + `application*.yml`.

---

## 1. 도메인 패키지의 표준 구조

각 도메인은 다음 구조로 정착:

```
com/bkurs/<domain>/
├─ controller/   # @RestController — 학생 / 강사 / 원장 / 마스터 별로 분리된 파일
├─ service/      # 트랜잭션 경계 (@Transactional) — 도메인 로직
├─ domain/       # JPA entity + Repository — 영속 모델 / 불변식
├─ dto/          # request/response — controller 경계 직렬화 타입
├─ enums/        # 도메인 상태 / 카테고리
├─ exception/    # 도메인 BusinessException 하위 — ErrorCode 매핑
├─ handler/      # (선택) 도메인 한정 @RestControllerAdvice — 글로벌 보다 우선
├─ gateway/      # (선택) 외부 SaaS 인터페이스 + impl swap (@ConditionalOnProperty)
├─ payment/      # (plan 도메인만) 결제 도메인 sub-aggregate
└─ bridge/       # (record 도메인만) 다른 도메인 entry 를 record 로 가져오는 어댑터
```

읽는 순서:
1. `controller/*Controller.java` 의 `@RequestMapping` 으로 외부 표면 (URL) 파악.
2. `dto/` 로 입출력 shape.
3. `service/` 로 트랜잭션 / 호출 그래프.
4. `domain/` 으로 entity / 불변식 / 관계.
5. `enums/` + `exception/` 으로 도메인 상태 모델 + 실패 케이스.
6. (외부 연동) `gateway/` 로 mock vs 실 impl 분기.

> 동일 도메인의 학생/강사/원장 컨트롤러가 분리된 이유는 **권한 별 표면 분리** + **role 별 path prefix 정착** 때문. 한 컨트롤러에 권한 분기 분기 코드가 섞이는 것을 피한다.

---

## 2. auth — 단일 Account + Role enum + Status enum

- entity: `Account` (`auth/domain/Account.java`) — 단일 테이블. role + status.
  - `Role { STUDENT_HIGH, STUDENT_MIDDLE, TEACHER, OWNER, MASTER }`
  - `AccountStatus { PENDING_CONNECT, ACTIVE, ... }` (학생만 가입 직후 PENDING_CONNECT)
- 분리:
  - `AccountService` — CRUD + bcrypt + 자격 검증
  - `AuthService` — 로그인 + refresh + logout 조립 (`@Transactional` 없음 — 위임)
  - `RefreshTokenService` — refresh 발급/회전/revoke + reuse 감지
  - `JwtTokenProvider` — access token 발급
  - `RefreshCookieFactory` — HttpOnly cookie 직렬화
- 결정 근거 javadoc: `decisions/decision-account-single-table-role-status`, `decisions/decision-auth-architecture-jwt-refresh` (BE 측 vault).

읽는 포인트:
- **role 검증의 책임은 service 가 아닌 SecurityConfig + @PreAuthorize.** 그러나 **signup endpoint 는 role 검증 부재** — public 으로 어떤 role 이든 자기발급 가능 ([[05-operational-risk-register#R1]]).
- AccountStatus 분기: 학생만 PENDING_CONNECT 로 시작 → connect (`/api/v1/auth/connect`) 후 ACTIVE. 강사 / 원장 / 마스터는 ACTIVE 로 시작 — 실 운영에서는 admin invite 후 활성화가 더 안전.

## 3. academy — 학원 + Membership + Connect + Transfer + Invite

- 컨트롤러가 가장 많은 도메인 (13). 책임 다음으로 분해:
  - **AcademyController** — 학원 자체 (코드 기반 조회)
  - **MyAcademyController** — 본인 소속 학원 (학생 / 강사 / 원장 시각)
  - **MembershipController** — 가입/탈퇴 CRUD
  - **OwnerMembershipQueueController** — 원장의 학생 승인 큐
  - **AcademyCodeController** — 학원 코드 발급/회전
  - **ConnectController** (`/api/v1/auth/connect`) — 학생이 학원 코드로 connect → PENDING_CONNECT → ACTIVE 전환
  - **TeacherConnectController** (`/api/v1/auth/teacher-connect`) — 강사 connect
  - **TeacherInviteController** — 강사 초대
  - **TeacherAssignmentController** — 강사 할당
  - **OwnerTransferFromController / OwnerTransferToController** — 원장 양도/양수
  - **OwnerStudentListController** — 원장의 학생 리스트
  - **StudentMembershipController** — 학생 시각 membership 조회

읽는 포인트:
- path prefix 가 **시각 (학생/강사/원장/마스터)** + **자원 위치 (academies/me, academies/my, students/me, teachers/me)** 의 직교 패턴. 새 endpoint 신설 시 동일 패턴 따라야 일관성 유지.
- Connect 흐름이 auth path 하위에 있는 점은 신경 쓸 만한 결정 — 가입 직후 흐름이지만 학원 도메인. 향후 `auth` 와 `academy` 도메인 경계가 모호해질 위험.

## 4. course — 수업 + 세션 + 출석

- 9 컨트롤러로 분해됨:
  - Owner 시각: `OwnerCourseController`, `OwnerSessionController`, `OwnerAttendanceController`
  - Teacher 시각: `TeacherCourseController`, `TeacherCourseUpdateController`, `TeacherSessionAttendanceController`, `TeacherStudentProfileController`
  - Student 시각: `StudentCourseController`, `StudentSessionAttendanceController`
- 도메인 entity: Course, CourseSession, Enrollment, Attendance (Flyway V12 / V13).

읽는 포인트:
- **수업 (Course) + 세션 (Session) + 출석 (Attendance)** 의 3-layer 분리. 출석은 강사가 입력 → 학생/원장이 view.
- `TeacherStudentProfileController` 는 학생 도메인이 아니라 course 도메인 — 강사가 본인 수업의 학생을 본다는 시각의 산물. 의도된 결정 (강사가 학원 전체 학생을 볼 수는 없게).

## 5. 학습 도메인 (record / statement / question / discussion / reading / performance / interview / simulation)

- 모두 **학생 + 강사 시각으로 분리된 2~3 컨트롤러** 패턴.
- record 만 `bridge/` 디렉터리 추가 — 다른 도메인 entry 를 record 로 가져오는 어댑터 (Flyway V23 — `alter_student_record_entry_source`).

읽는 포인트:
- record 가 모든 도메인을 모으는 **집합 도메인** 인 점이 중요. 새 도메인 (suhaeng / saenggibu / concept 등) 추가 시 record 의 source enum 도 함께 늘어남.
- 권한: 학생은 본인 데이터만, 강사는 본인 수업의 학생만, 원장은 본인 학원의 학생만. URL prefix 가 이미 그 권한을 표현.

## 6. suhaeng / saenggibu / concept

- **suhaeng (수행 평가)** — 학생 + 강사 + 원장 시각 분리. round 별 status.
- **saenggibu (생기부)** — 학생 + 강사 (academy 단위) + 원장 (lock). lock controller 는 원장 전용.
- **concept (진로 진단)** — 학생 + 강사 시각. **FE 는 아직 Supabase 직접 호출 잔류** ([[02-fe-flow-inventory#5]]).

읽는 포인트:
- saenggibu lock 은 원장만 해제 — 권한 분리가 path 만으로 명확.
- concept 는 BE 완료 + FE swap 만 남은 상태 — cutover 우선순위 P0.

## 7. admission — past-question / major-question

- 학생 시각 (`/students/me/...`) + master 시각 (`/master/...`) 만. 강사/원장은 없음 (운영 콘텐츠가 본사 단일).
- FE Master `Questions.tsx` 는 이미 BE 연동 완료. 향후 passage tab 추가 시 동일 도메인에 controller 신설 패턴 그대로.

## 8. plan / payment / subscription

- `PlanCatalogController` — 공개 (`/api/v1/plans`)
- `AdminPlanController` — admin write (`/api/v1/admin/plans`)
- `SubscriptionController` — 본인 학원 구독 (`/api/v1/academies/me/subscription`)
- `PaymentController` — 결제 처리

도메인 경계:
- `plan/payment/PaymentService` 가 `PaymentGateway` 인터페이스를 통해 결제. impl 은 mock or PortOne (gateway swap).
- **MockPaymentGateway 는 `@ConditionalOnProperty` 없이 `@Component`** — 잠재 버그 ([[05-operational-risk-register#R4]]).

## 9. file / pdf

- `FileStorageController` — presigned URL 발급만. 클라가 직접 PUT.
- `pdf/StudentRecordPdfController` — record PDF 생성 — `PdfGenerator` gateway (mock 만 존재. 실 impl 없음).
- 영상 / 이미지 / PDF 모두 S3 cutover 가 한 번에 묶임 — `bkurs.external.file-storage.impl=s3` + `pdf.impl=real` (B17/B23).

## 10. notification / ai / stt — 모두 gateway swap

- `NotificationService` — mock (default) / sendgrid (`@ConditionalOnProperty`). 비동기 dispatcher 있음 (`AsyncNotificationDispatcher`).
- `AiAssistant` — mock (default) / anthropic (Claude).
- `SttService` — mock (default) / clova.

읽는 포인트:
- 인터페이스 + impl swap 이 **표준** — 본사 도메인 외 모든 외부 의존은 이 패턴.
- impl 키는 `bkurs.external.<service>.impl` 로 통일.
- mock 의 `matchIfMissing=true` 가 표준이지만 **MockPaymentGateway 만 누락** — 통일 권고.

## 11. messaging / batch

- `common/messaging/`:
  - `MessagingPublisher` 인터페이스 + `InMemoryMessagingPublisher` (default) / `RabbitMessagingPublisher` (`@ConditionalOnProperty`).
  - `RabbitConfig` — exchange / queue / DLQ.
- `batch/`:
  - Spring Batch + `OverduePaymentAlertJobConfig` — 미납 알림 batch.
  - `ScheduledBatchLauncher` — cron 트리거.
  - 로컬 profile 은 `spring.batch.job.enabled: false` — 자동 실행 차단.

읽는 포인트:
- prod 진입 시 batch enabled 여부 + DLQ 정책 점검 필요. 별도 결정 ([[07-open-questions#Q5]]).

---

## 12. 도메인 경계가 모호한 지점

확인 / 추측 구분:
- **확인:** Connect 가 auth 와 academy 의 경계에 있음 — path 는 auth, 도메인 코드는 academy. 향후 confusion 우려.
- **확인:** record 가 다른 도메인 entry 를 가져오는 bridge — record 가 비대해질 수 있음. record 가 단순 view layer 인지 / 영속 layer 인지 명확히 (read source 명세).
- **확인:** `OwnerStudentProfileController` (academy 도메인의 view) vs `StudentProfileController` (student 도메인) vs `TeacherStudentProfileController` (course 도메인) 가 3개 도메인에 흩어져 있음 — 의도된 시각 분리지만 신규 개발자에게 학습 부담.
- **추측 (확인 필요):** `FileStorageController` 가 `file` 도메인 — 이후 pdf / image / 영상이 같은 controller 로 모이거나 분리될지. 현재는 presigned URL 발급만이므로 OK.

## 13. 합치거나 분리하면 좋을 지점 (제안 — 단정 X)

- **합치기 후보:** auth/Connect + academy/Connect 의 path 정합 (`/api/v1/auth/connect` 가 academy 도메인 컨트롤러). 향후 `/api/v1/auth/*` 는 인증 자체만, `/api/v1/academies/connect` 로 옮기는 것이 도메인 일관성에 더 가깝다 (FE breaking change 비용은 따로 평가).
- **분리 후보:** Master role 의 모든 endpoint 가 `/api/v1/master/*` 로 단일 prefix — 현재는 admission 의 master 만. 향후 staff / academies / billing / data 등 신설 시 동일 prefix 정착이 일관.
- **분리 후보:** FE Master localStorage 가 운영 데이터를 흉내 — BE Master 도메인 신설 시 데이터 분리 / 권한 분리 / audit 분리 동시 결정.

> 본 섹션은 **읽는 법** 의 일부 — 즉시 변경 권고가 아님. 본격 리팩토링은 별도 PR.
