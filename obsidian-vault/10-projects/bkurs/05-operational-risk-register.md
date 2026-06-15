---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, risk, deployment, security]
related:
  - "[[00-index]]"
  - "[[02-fe-flow-inventory]]"
  - "[[03-be-api-contract]]"
  - "[[06-deployment-readiness-roadmap]]"
  - "[[07-open-questions]]"
---

# 05 · 운영 위험 레지스터

> Severity scale:
> - **Critical** — 운영 데이터 / 권한 / 자격 정보 직접 위협, 즉시 해결 필요
> - **High** — 운영 거짓 데이터 / 외부 API 표면 / 배포 차단 → 1~2주 안 해결 권고
> - **Medium** — 운영 정합 / 신뢰 / 추적 → release window 단위 해결 권고
> - **Low** — 이미 mitigated 또는 다음 cycle 에서 정리

각 항목은 [[06-deployment-readiness-roadmap]] 의 단계와 연동됨.

---

## R1 · Public signup 의 role 화이트리스트 부재
- **Severity:** Critical
- **Evidence:**
  - `src/main/java/com/bkurs/auth/dto/request/SignupRequest.java` — `role: Role` (`@NotNull` 만, enum 자체 검증 없음). `STUDENT_HIGH / STUDENT_MIDDLE / TEACHER / OWNER / MASTER` 5개 중 어떤 값이든 허용.
  - `src/main/java/com/bkurs/auth/service/AccountService.signup(...)` — role 별 분기는 status (학생 PENDING_CONNECT vs 그 외 ACTIVE) 만.
  - `src/main/java/com/bkurs/common/config/SecurityConfig.java:52` — `/api/v1/auth/signup` `permitAll`.
- **Impact:** 어떤 인터넷 사용자나 `POST /api/v1/auth/signup` 으로 `role=MASTER` 또는 `role=OWNER` 계정 생성 가능 → 인증 직후 `ACTIVE` 상태 → `/api/v1/master/*` 또는 `/api/v1/academies/me/*` 접근.
- **Mitigation 옵션:**
  1. signup DTO 에서 role 화이트리스트 (`STUDENT_HIGH`, `STUDENT_MIDDLE` 만 허용). 외 역할은 invite/bootstrap 흐름 신설.
  2. `AccountService.signup` 에서 role 검증 후 다른 role 은 `BusinessException` (AUTH-4001 or 별도 코드).
  3. 강사 / 원장 invite 는 별도 endpoint (`TeacherInviteController` 가 이미 있음 — 이 흐름으로 강제).
  4. MASTER bootstrap 은 별도 admin CLI / SQL seed / `ApplicationRunner` 로만.
- **Owner:** BE
- **Priority:** P0 (운영 전 차단 필수)

## R2 · Master role FE 가 mock / localStorage 위주
- **Severity:** High
- **Evidence:**
  - `/master/notices`, `/master/staff` → `localStorage` (`master_notices`, `master_staff`). `useNotices.ts`, `useStaff.ts`.
  - `/master/academies`, `/master/billing`, `/master/data`, `/master/audit`, `/master/dashboard` → 정적 mock (`_data/*-data.ts`).
  - `/master/lessons` → mock + Supabase storage placeholder (`useUploadFlow.ts`).
  - `/master/questions` 만 BE 연동 (`Questions.tsx` + `pastQuestionApi` / `majorQuestionApi`).
- **Impact:** Master role 운영 노출 시 9/10 페이지가 거짓 데이터. 본사 관점에서 학원 현황 / 매출 / 직원 / 감사 로그 신뢰 불가.
- **Mitigation 옵션:**
  1. Master 표면을 "internal preview" / "read-only" banner 로 표시 → 운영 release 에서 의사결정 데이터로 사용 금지.
  2. 또는 운영 release 에서 비활성 (router 차단) — 본사 사용자가 해당 화면 자체에 들어오지 못하게.
  3. 본사 도메인 BE endpoint 신설 후 점진 swap (academies write / billing aggregate / staff / audit).
- **Owner:** Product + FE + BE
- **Priority:** P0 (운영 노출 전 결정 필수)

## R3 · Supabase 직접 호출 (concept + 영상 storage placeholder)
- **Severity:** High
- **Evidence:**
  - concept 페이지 6 파일 (`high-student/_pages/concept/{CareerConcept,ConceptDiagnosis,ConceptResult}.tsx`, `middle-student/_pages/concept/...`) 이 `import { supabase } from '@/lib/supabase'` + `supabase.from('student_concept').select(...)` 직접 호출.
  - `src/pages/master/_pages/lessons/useUploadFlow.ts` — `supabase.storage.from('lessons').upload(...)` placeholder.
  - `admin/_layout/AdminLayout.tsx`, `AdminSidebar.tsx` 등 일부 admin 화면 import 만 있고 placeholder 주석.
- **Impact:**
  - 권한 분리 불명확 — Supabase RLS / anon key 가 학원 단위 분리되지 않음.
  - `VITE_SUPABASE_ANON_KEY` 가 FE 번들에 포함 (Vite 정의상 클라이언트 노출).
  - BE 인증 / 권한 / audit 와 무관한 데이터 경로가 남아 있음.
  - cutover 누락 시 prod 운영 중 reads / writes 가 BE 우회.
- **Mitigation:**
  - concept 6 파일 P0 cutover — `useConceptDiagnosis` hook 이 이미 BE `conceptApi` 호출. 페이지에 `supabase.from(...)` 만 hook 호출로 swap.
  - 영상 storage placeholder 는 S3 cutover (R7) 와 같이 처리.
  - admin placeholder import 는 unused — cleanup PR.
  - cutover 후 `src/lib/supabase.ts` + `VITE_SUPABASE_*` 제거 또는 빈 stub 으로.
- **Owner:** FE + BE (concept hook 검증)
- **Priority:** P0 (cutover) / P1 (S3)

## R4 · MockPaymentGateway 의 `@ConditionalOnProperty` 누락
- **Severity:** Medium
- **Evidence:**
  - `src/main/java/com/bkurs/plan/gateway/MockPaymentGateway.java` — `@Component` 만 부착, `@ConditionalOnProperty` 없음.
  - `src/main/java/com/bkurs/plan/gateway/PortOnePaymentGateway.java:28-29` — `@Component` + `@ConditionalOnProperty(name = "bkurs.external.payment.impl", havingValue = "portone")`.
  - 다른 6 gateway (`MockFileStorage`, `MockNotificationService`, `MockAiAssistant`, `MockSttService`, `MockPdfGenerator`, `MockMessagingPublisher`-equivalent) 는 모두 `@ConditionalOnProperty(... havingValue = "mock", matchIfMissing = true)` 패턴.
- **Impact:**
  - `bkurs.external.payment.impl=portone` 시 `MockPaymentGateway` + `PortOnePaymentGateway` 둘 다 `PaymentGateway` 빈 등록 → `PaymentService.gateway` 단일 주입 충돌 → `NoUniqueBeanDefinitionException` 으로 startup fail 가능.
  - 운영 진입 시 결제 활성화 첫 배포에서 차단.
- **Mitigation:**
  - `MockPaymentGateway` 에 `@ConditionalOnProperty(name = "bkurs.external.payment.impl", havingValue = "mock", matchIfMissing = true)` 부착.
  - 또는 `@Primary` 로 Mock 을 명시 (덜 권고 — impl swap 의도 깨짐).
- **Owner:** BE
- **Priority:** P1 (결제 cutover 전 차단)

## R5 · Swagger / OpenAPI permitAll
- **Severity:** Medium-High
- **Evidence:** `SecurityConfig.filterChain:59-65` — `/swagger-ui.html`, `/swagger-ui/**`, `/v3/api-docs`, `/v3/api-docs/**`, `/swagger-resources/**` `permitAll`.
- **Impact:** Production 노출 시 70+ endpoint 의 path / dto / 예시 데이터 / role 분리 전체가 외부에 공개. 인증된 사용자라도 본인 권한 외 endpoint 까지 발견 가능.
- **Mitigation:**
  - prod profile 에서 swagger / openapi 비활성:
    - `application-prod.yml` 에 `springdoc.swagger-ui.enabled: false` + `springdoc.api-docs.enabled: false`.
    - 또는 별도 `@Profile("!prod")` 로 `OpenApiConfig` 제한.
  - 또는 ALB / nginx 레벨에서 path 차단 (IP allowlist + basic auth).
  - dev/stg 만 노출 + bearer JWT 필수로 전환.
- **Owner:** BE + Infra
- **Priority:** P0 (prod 노출 전)

## R6 · CI/CD deploy step 미완성
- **Severity:** Medium-High
- **Evidence:**
  - `bkurs-be/.github/workflows/ci.yml` — build + test (MySQL service container). 정상.
  - `bkurs-be/.github/workflows/deploy-dev.yml` — GHCR image build 까지. 이후 deploy step (ECS / SSH / k8s) 전부 주석 처리 (`# deploy:` 이하).
  - `bkurs-fe/.github/workflows/ci.yml` — lint / format / typecheck / build (test 없음). deploy workflow 없음.
- **Impact:**
  - 실 배포 자동화 부재 — 수동 release 의존.
  - FE 에 test job 부재 — 회귀 안전망 누락.
  - prod blue/green 전환 시 절차 없음.
- **Mitigation:**
  - deploy 옵션 결정 ([[07-open-questions#Q4]]): ECS / EC2+SSH / k8s 중 하나.
  - 결정 후 `deploy-dev.yml` 활성화 + `deploy-prod.yml` 신설 (수동 trigger + 승인 게이트).
  - FE deploy workflow (S3+CloudFront or 다른 호스팅) 신설.
  - FE 에 jest / vitest 회귀 + critical-path e2e (Playwright) 신설 검토.
- **Owner:** Infra + FE/BE
- **Priority:** P1

## R7 · S3 / Storage 미구현 — placeholder
- **Severity:** Medium-High
- **Evidence:**
  - `src/main/java/com/bkurs/file/gateway/MockFileStorage.java` 만 존재 — `S3FileStorage` 등 실 impl 없음.
  - `bkurs.external.file-storage.impl=s3` 환경에서 동작하는 빈 없음 → startup fail 또는 BeanCreationException.
  - FE `useUploadFlow.ts` 의 `simulateUpload` 가 실 업로드 X.
  - PDF 도 `MockPdfGenerator` 만 — 실 impl 없음.
- **Impact:**
  - 영상 / 이미지 / 자료 업로드가 mock — 실 운영 시 파일 전혀 저장되지 않음.
  - record PDF 생성도 mock — 학생 record 의 PDF 출력이 빈 placeholder.
- **Mitigation:**
  - B17: `S3FileStorage` impl + presigned URL 발급 + bucket 정책 + lifecycle.
  - B23: `PdfGenerator` 실 impl (예: openhtmltopdf / wkhtmltopdf / iText / 외부 SaaS).
  - 운영 전까지는 파일 업로드 UI 비활성 또는 read-only 결정 가능 ([[07-open-questions#Q4]]).
- **Owner:** BE + Infra
- **Priority:** P1 (concept S3 의존 cutover 후 / lesson 영상 의존)

## R8 · prod 자격 정보 부재 / secret manager 통합
- **Severity:** Medium
- **Evidence:**
  - `application-prod.yml` — 모든 secret 이 `${VAR:?...required}` — 누락 시 startup fail (좋음).
  - 그러나 secret 주입 경로 (AWS SSM / Vault / k8s Secret) 가 코드/문서로 강제되지 않음.
  - `.env.example` 에 모든 외부 SaaS 키 placeholder 만.
- **Impact:** prod 배포 시 secret 누락 → startup fail → 다행이지만, secret 회전 / 격리 / 감사 미정.
- **Mitigation:**
  - secret manager 결정 ([[07-open-questions#Q6]]) + IAM 권한 정책 + 로컬 dev 와의 분리 명시.
  - prod runbook 작성 (어떤 secret 이 어느 store 에 있는지 매핑).
- **Owner:** Infra
- **Priority:** P1

## R9 · CORS / cookie / 도메인 정합
- **Severity:** Low-Medium
- **Evidence:**
  - `application-prod.yml` `cors.allowed-origin-patterns: ${FRONTEND_URL:?...}` — 단일 origin.
  - `jwt.refresh-cookie-secure: true` (prod / dev 모두) — HTTPS 강제.
  - `SameSite=Lax` — 같은 site cross-subdomain 만 허용.
  - `allowCredentials=true` + patterns 사용 — Spring 6 / 모던 브라우저 호환.
- **Impact:**
  - staging + prod 동시 운영 시 multi-origin 필요 (현재 단일 env var).
  - FE 가 `api.bkurs.com` 별 도메인 분리 시 SameSite=Lax 가 cross-site 로 분류될 수 있음.
- **Mitigation:**
  - `FRONTEND_URL` 을 다중 origin 가능하게 (`List<String>` 환경 변수).
  - FE / BE 도메인 정책 결정 ([[07-open-questions#Q2]]): same-site (`/api/*` CloudFront 라우팅) vs cross-site (`api.bkurs.com`). cross-site 면 `SameSite=None; Secure` 로 변경.
- **Owner:** Infra + BE
- **Priority:** P1

## R10 · Flyway baseline 정책 미정
- **Severity:** Low-Medium
- **Evidence:** `application-prod.yml` — `flyway.baseline-on-migrate: false`, `validate-on-migrate: true`.
- **Impact:**
  - prod 첫 배포 시 빈 DB 면 OK (V1 부터 실행). 그러나 기존 DB 위에 올릴 일이 생기면 baseline 결정 필요.
  - `validate-on-migrate=true` — 스크립트 checksum 변경 시 startup fail. 운영 hotfix 시 V`n+1` 신규 마이그레이션 강제 (좋음).
- **Mitigation:**
  - prod 첫 배포 runbook 에 "빈 DB 가 보장될 것" 명시. 아니면 `baselineOnMigrate=true` + `baselineVersion=0` 옵션 검토.
  - 마이그레이션 정책 SSoT 작성 (변경 금지 / 신규 V`n+1` 만 허용).
- **Owner:** BE
- **Priority:** P1

## R11 · 외부 SaaS mock 운영 잔류 위험
- **Severity:** Medium
- **Evidence:**
  - `application-dev.yml` — 외부 SaaS 6개 모두 `mock` default.
  - `docker-compose.dev.yml` — `*_IMPL: mock` 강제.
  - prod `application-prod.yml` 에는 `bkurs.external.*` 명시 없음 — env 로만 결정. mock 으로 떠도 startup 정상.
- **Impact:** prod 에서 결제 / 메일 / AI / STT / PDF 가 mock 으로 동작 — 운영 거짓 결과.
- **Mitigation:**
  - prod startup 시 `mock` 잔류 감지 + 경고 / fail (BeanPostProcessor + 환경 검증).
  - 또는 prod profile 에 `bkurs.external.*.impl=...` 명시 강제 + CI 검증.
- **Owner:** BE
- **Priority:** P1

## R12 · FE `VITE_SUPABASE_*` 가 운영 빌드에 포함
- **Severity:** Low (Supabase 잔류시 — Medium)
- **Evidence:**
  - `bkurs-fe/.github/workflows/ci.yml:50-53` — build 시 `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` 주입 (없으면 placeholder).
  - `src/lib/env.ts` 가 빌드 시점 inline.
- **Impact:** anon key 가 dist 번들에 포함 — Supabase prod 키가 노출. anon key 자체는 RLS 정책으로 보호되어야 하나, RLS 정책이 학원 단위 분리되지 않음 → 데이터 읽기 가능성.
- **Mitigation:**
  - cutover 완료 후 `VITE_SUPABASE_*` 제거 + `supabase.ts` 삭제.
  - 잔류 동안: Supabase prod 프로젝트 분리 + RLS 강화 + anon key rotate 검토.
- **Owner:** FE + Supabase admin
- **Priority:** P1 (cutover 완료 시 자연스럽게 해결)

## R13 · prod actuator endpoint 노출
- **Severity:** Low-Medium
- **Evidence:** `application.yml` — `management.endpoints.web.exposure.include: health, info` (good — 추가 endpoint 없음). `endpoint.health.show-details: always` — 인증 미적용 (`SecurityConfig` 의 permitAll 에 `/actuator/health` 포함).
- **Impact:** `show-details: always` + permitAll → DB / disk / 의존성 상세 노출. 정보 누설 약하지만 attack surface 키움.
- **Mitigation:**
  - prod 에서 `endpoint.health.show-details: when_authorized` + actuator 별도 management port + IP allowlist.
- **Owner:** BE + Infra
- **Priority:** P2

## R14 · FE CI 에 test job 부재
- **Severity:** Low-Medium
- **Evidence:** `bkurs-fe/.github/workflows/ci.yml` — lint / format / typecheck / build 만.
- **Impact:** 회귀 안전망 부재 — 핸드오버 시 일관성 보장 어려움.
- **Mitigation:**
  - vitest + React Testing Library 단위 테스트.
  - apiClient mock + envelope unwrap 테스트.
  - critical-path Playwright e2e (login → connect → roadmap 등).
- **Owner:** FE
- **Priority:** P2

## R15 · Blue/Green 전환 부재
- **Severity:** Medium
- **Evidence:** Dockerfile + GHCR push 까지. ALB / target group / route53 weighted routing 등 인프라 부재.
- **Impact:** prod release 시 downtime / rollback 절차 없음.
- **Mitigation:**
  - 인프라 결정 후 ECS blue/green (CodeDeploy) 또는 k8s rolling/blue-green.
  - smoke test → 트래픽 점진 swap → rollback 기준 ([[06-deployment-readiness-roadmap]] §4).
- **Owner:** Infra
- **Priority:** P2 (P1 staging 안정 후)

---

## 요약 매트릭스

| ID | Risk | Severity | Owner | Priority |
| --- | --- | --- | --- | --- |
| R1 | Public signup role 화이트리스트 부재 | Critical | BE | P0 |
| R2 | Master FE mock/localStorage 위주 | High | Product+FE+BE | P0 |
| R3 | Supabase 직접 호출 잔류 | High | FE+BE | P0 (concept) / P1 (storage) |
| R5 | Swagger permitAll | Medium-High | BE+Infra | P0 |
| R4 | MockPaymentGateway 조건 누락 | Medium | BE | P1 |
| R6 | CI/CD deploy step 미완성 | Medium-High | Infra+FE+BE | P1 |
| R7 | S3/Storage 미구현 | Medium-High | BE+Infra | P1 |
| R8 | prod secret manager 미정 | Medium | Infra | P1 |
| R9 | CORS/cookie/도메인 정합 | Low-Medium | Infra+BE | P1 |
| R10 | Flyway baseline 정책 미정 | Low-Medium | BE | P1 |
| R11 | 외부 SaaS mock 운영 잔류 | Medium | BE | P1 |
| R12 | FE Supabase 키 번들 포함 | Low | FE | P1 (cutover 후 자연 해결) |
| R13 | actuator details always | Low-Medium | BE+Infra | P2 |
| R14 | FE CI test job 부재 | Low-Medium | FE | P2 |
| R15 | Blue/Green 부재 | Medium | Infra | P2 |
