---
title: "B4A — 학원 정보 (phone/address/logo/website) + 학생·강사 조회 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b4-payment
created_at: 2026-05-20T08:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-academy-info-fields]]"
  - "[[task-log-b3c-announcement]]"
tags: [task-log, academy, info-fields, b4a]
---

# 작업

GitHub issue [#15](https://github.com/seum-running/bkurs-be/issues/15) — 학원 정보 보강 + 학생·강사 조회.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 데이터 모델 | academy 테이블에 nullable 4 컬럼 직접 추가 | [[../decisions/decision-academy-info-fields]] |
| PATCH null 처리 | entity.updateInfo 가 null/blank 무시 — 부분 업데이트 idempotent | |
| 조회 권한 | AcademyMembershipResolver 재사용 (학원장/학생/강사 모두) | B3C resolver 그대로 |
| 학원 미가입 응답 | 404 (목록은 빈 배열, 단일 객체는 404 가 자연스러움) | |
| URL 검증 | @Size + @Pattern (^https?://) — false positive 회피 | 엄격 검증은 클라이언트 |

# 진행

## 2026-05-20

### 1. branch / issue
- B3C 머지 대기 중 stacked PR — base=feature/issue-13-b3c-announcement
- issue #15 + decision 노트 (4 대안 × 2)

### 2. 코드 (commit 단위)
- C1: V7 ALTER + Academy entity 4 필드 + updateInfo invariant
- C2: UpdateAcademyRequest / AcademyResponse 확장 + AcademyManagementService.updateInfoMine + AcademyController PATCH 확장 + 신규 MyAcademyController GET /my/academy
- C3: GlobalValidationExceptionHandler (회수 함정) + AcademyInfoApiTest 8 + CHANGELOG B4A

### 3. PR
- PR [#16](https://github.com/seum-running/bkurs-be/pull/16) (feature/issue-15 → feature/issue-13-b3c-announcement, stacked)
- 회귀 **62/62 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| academy 테이블 nullable 4 컬럼 (academy_info 분리 X) | 4 필드는 작음 + JOIN 비용 없음 + PATCH endpoint 단일. 정규화 비용 > 단순성 이득의 분기점은 아님. | 컬럼 10+ 누적 시 academy_metadata 또는 academy_settings 분리 검토. |
| 모든 필드 nullable | 학원 생성 시점에 정보 다 없음 → 점진 채움 UX. NOT NULL 이면 학원 생성 자체 불가. | 필수 필드는 (B4+) 결제 가입 시점에 별도 검증. |
| `updateInfo` null/blank 무시 invariant | 클라가 "변경 안 함" = null 전송, "지우기" = ? (의도 불명) — 안전한 default 는 null 무시. | "지우기" 명시 endpoint (DELETE /me/info/{field}) 필요 시 별도 추가. |
| 빈 문자열도 null 처리 | null vs "" 구분이 클라에 부담. 둘 다 무시로 통일. | 정밀 제어 필요 시 PATCH JSON null 명시 vs absent 구분 (JsonNullable) 검토. |
| AcademyMembershipResolver 재사용 (announcement 패키지) | B3C 의 resolver 가 학원장/학생/강사 통합 — B4A 도 같은 분기 필요. 단일 진입점. | resolver 가 announcement 패키지에 있는 게 어색 — B5+ 에서 com.bkurs.access 같은 cross-cutting 패키지로 이동 검토. |
| 학원 미가입 → 404 (목록 endpoint 와 다른 정책) | 단일 객체 GET 에서 빈 응답은 의미 모호 (200 + 빈 body 가능). 404 가 REST 관례 + 클라 분기 단순. | 일부 운영자는 200 + null body 선호 — 본 PR 은 404 통일. |
| GET `/my/academy` 모든 인증 사용자 (role 무관) | OWNER 도 본인 학원 조회 가능 (편의). Resolver 가 OWNER → 본인 학원 ID 반환. | 향후 role 별 응답 분기 (학원장 응답엔 결제 정보 / 학생 응답엔 안 보임) 시 분리. |
| `GlobalValidationExceptionHandler` (회수) | @Valid 실패 시 Spring ERROR dispatch /error 가 SecurityConfig permitAll 부재로 401 → 명시 핸들러로 400 직접 응답. /error permitAll 보다 안전 (인증 필요 endpoint 의 500 응답 노출 차단). | 모든 도메인에 영향 — 향후 모든 controller 의 @Valid 실패 통일. |
| @Order HIGHEST_PRECEDENCE | 도메인 한정 핸들러 (auth/academy/announcement) 보다 먼저 매칭. validation 은 가장 기본적 layer. | 도메인 핸들러가 MethodArgumentNotValidException 직접 매핑하지 않는 한 충돌 없음. |
| 도메인 한정 @RestControllerAdvice (basePackages) | 도메인별 응답 envelope 차별화 가능. Global handler 와 같이 쓸 때 @Order 로 우선순위. | global handler 가 처리하는 예외와 충돌 안 하도록 도메인은 도메인 specific 예외만. |

# 다음 단계 (예정)

- B2A → ... → B3C → B4A 순으로 사용자 manual merge → CI 자동 실행
- B4B 시작 — 학원 플랜 (Plan + AcademyPlanSubscription)
- B4C — 결제 연동 (외부 게이트웨이 / mock)
- B4D — 학원장 이관 (owner 변경 endpoint + 감사 로그)

# 회수 / pitfall (누적)

- **@Valid 실패 시 401 응답 함정** — Spring ERROR dispatch /error 경로가 SecurityConfig permitAll 부재로 인증 필터에 가로채짐. 디버깅 1 cycle 소요. vault `20-areas/backend/spring/java-spring/pitfalls/error-dispatch-permitAll-trap.md` 신설.

# 참고

- [[../decisions/decision-academy-info-fields]]
- [[task-log-b3c-announcement]]
- [[../../../20-areas/backend/spring/java-spring/pitfalls/error-dispatch-permitAll-trap]]
