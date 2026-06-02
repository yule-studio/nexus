---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, deployment, roadmap, aws, staging, blue-green]
related:
  - "[[00-index]]"
  - "[[05-operational-risk-register]]"
  - "[[07-open-questions]]"
---

# 06 · 배포 준비 로드맵 (staging → prod)

> 본 문서는 **AWS 배포 가정** 하에 작성. 실제 인프라 / 클라우드 선택은 [[07-open-questions#Q4]] 의사결정 후 갱신.
>
> 모든 단계는 "지금 할 일 (즉시)" / "나중에 할 일 (cutover 단계)" 로 구분.

---

## 1. 단계 요약

| 단계 | 목표 | 입구 조건 | 출구 조건 |
| --- | --- | --- | --- |
| P0 — 구조 위험 차단 | 운영 노출 시 회복 불가한 위험 제거 | 현재 | R1/R2/R3 (concept)/R5 mitigated |
| P1 — staging 안정화 | dev/stg 환경 실제 트래픽 흐름 | P0 완료 | 모든 BE 호출 e2e OK / mock 잔류 없음 / smoke 통과 |
| P2 — 외부 SaaS 활성화 | S3 / Payment / Email / AI / STT 실 impl | P1 완료 | 단일 환경 ENV swap 만으로 전환 가능 |
| P3 — prod blue/green | 무중단 배포 + rollback | P2 완료 | blue/green CodeDeploy or k8s 절차 정착 |
| P4 — 운영 / 회귀 / 관측 | 로그 / 메트릭 / Sentry / batch 정책 | P3 완료 | 24/7 운영 가능 |

본 로드맵의 risk ID 는 [[05-operational-risk-register]] 와 동기.

---

## 2. P0 — 구조 위험 차단 (즉시)

목표: **운영 노출 시 회복 불가한 위험만 차단**. 이 단계 안에서는 인프라 구축은 하지 않음.

### 2.1 BE (코드)
- [ ] R1: signup DTO 의 role 화이트리스트 — `STUDENT_HIGH`, `STUDENT_MIDDLE` 만 허용. 다른 role 은 invite/bootstrap 으로.
- [ ] R5: prod profile 에서 swagger 비활성 (`springdoc.swagger-ui.enabled: false` 추가) 또는 IP allowlist.
- [ ] R4: `MockPaymentGateway` 에 `@ConditionalOnProperty(... matchIfMissing=true)` 부착 (사전 안전망).
- [ ] R11: prod 시작 시 `bkurs.external.*.impl=mock` 잔류 감지 → 경고 로그 또는 fail. `Environment` 빈 검사하는 `ApplicationRunner`.
- [ ] R13: `application-prod.yml` 에 `management.endpoint.health.show-details: when_authorized` + management port 분리 검토.

### 2.2 FE (코드)
- [ ] R3 concept cutover (6 파일): `CareerConcept.tsx`, `ConceptDiagnosis.tsx`, `ConceptResult.tsx` × {high-student, middle-student} — `supabase.from('student_concept')` → `useConceptDiagnosis()` 또는 `conceptApi` 직접.
- [ ] R3 admin placeholder import cleanup (`AdminLayout.tsx`, `AdminSidebar.tsx`, 일부 tab `*Tab.tsx`).
- [ ] R2 Master role 운영 노출 정책 결정:
  - 옵션 A: `/master/*` 전체 router 차단 (운영 release 에서 redirect).
  - 옵션 B: 페이지별 banner ("아직 베타 — 데이터는 mock 입니다") + read-only.
- [ ] R14: FE vitest + 로그인 / connect / record list 등 critical-path 단위 테스트 신설 (P0 의 boundary 일 뿐).

### 2.3 검증
- 자동:
  - BE: `./gradlew clean check` 통과.
  - FE: `npm run lint && npm run typecheck && npm run build`.
- 수동:
  - signup curl 로 `role=MASTER` / `role=OWNER` 시도 → 400 응답 확인.
  - prod profile 로 boot → `/swagger-ui.html` 404/403 확인.
  - concept 화면에서 network 탭에 `supabase` 호출 없음 확인.

---

## 3. P1 — staging 안정화

목표: dev/stg 환경에서 실제 트래픽 흐름 검증. 외부 SaaS 는 아직 mock 가능 / 일부 stg 환경에서 실 SaaS 시도.

### 3.1 인프라 (staging — AWS 가정)
- [ ] VPC + 1 AZ (dev) / 2 AZ (stg) — 비용 절감.
- [ ] RDS (MySQL 9) 또는 Aurora MySQL — dev/stg 분리.
- [ ] ECR repo (`bkurs-be`) — 현재 GHCR 에서 이동 또는 병행 ([[07-open-questions#Q7]]).
- [ ] CloudFront + S3 (`bkurs-fe`) — dist 호스팅.
- [ ] ALB (BE) → ECS Fargate (또는 EC2) — 단일 서비스.
- [ ] SSM Parameter Store (secret manager) — JWT_SECRET / DB_PASSWORD / FRONTEND_URL / AWS keys.
- [ ] Route53 — `stg.bkurs.com` (FE), `stg-api.bkurs.com` 또는 same-site (`stg.bkurs.com/api/*`).
- [ ] CloudWatch Logs / alarms (BE log group + 5xx 알람).

### 3.2 BE 코드 / yml
- [ ] `application-stg.yml` 신설 (또는 prod 와 dev 차이만 env 로 결정).
- [ ] R9: `cors.allowed-origin-patterns` 를 환경 변수 multi-origin 지원 (`List<String>`).
- [ ] R10: Flyway baseline 정책 명문화 (stg 첫 배포 시 빈 DB 보장 → `baseline-on-migrate: false`).
- [ ] R8: secret 주입 경로 명시 — 각 env 변수가 어디서 오는지 README + runbook.

### 3.3 FE 코드 / 환경
- [ ] `VITE_API_BASE_URL` / `VITE_BE_ORIGIN` env 분리 (dev / stg / prod).
- [ ] R12 cutover 후 Supabase 키 제거 또는 stg-only 임시 잔류.
- [ ] CI: stg secret 으로 build artifact + S3 deploy.

### 3.4 CI/CD
- [ ] R6: `deploy-dev.yml` 의 deploy step 활성화 (ECS / SSH / k8s 중 1).
- [ ] `deploy-stg.yml` 신설 — workflow_dispatch + 환경 보호 룰 (필수 reviewer 2명).
- [ ] FE deploy workflow (S3 sync + CloudFront invalidation).

### 3.5 smoke test (수동 / 자동)
스테이징 ALB 도달 후 다음 시나리오 통과:

1. **회원가입 → connect → roadmap 노출**
   - `POST /api/v1/auth/signup` (role=STUDENT_HIGH)
   - 학원 코드 입력 → `/api/v1/auth/connect`
   - 200 응답 + status ACTIVE → `/high-student/roadmap` 로딩
2. **로그인 → /me → 학원 정보**
   - `POST /api/v1/auth/login` → access + cookie
   - `GET /api/v1/auth/me` → role/status
   - `GET /api/v1/academies/my` → 학원 정보
3. **record 1건 작성**
   - `POST /api/v1/students/me/record-entries` (또는 도메인 별 entry)
   - `GET /api/v1/students/me/records` 로 노출 확인
4. **logout → cookie revoke**
   - `POST /api/v1/auth/logout` → 204 + Set-Cookie max-age=0
   - 후속 401 확인
5. **invalid signup**
   - `role=MASTER` 시도 → 400 (R1 차단)
6. **Swagger 404**
   - `/swagger-ui.html` prod profile 에서 404/403 확인 (R5)
7. **CORS 차단**
   - 알 수 없는 origin → CORS rejected
8. **invalid token**
   - 만료 access → 401 → refresh 자동 → 원 요청 1회 재시도 (interceptor)
9. **DB 마이그레이션**
   - 컨테이너 재시작 시 V1~V29 적용 완료. validate 통과.
10. **CloudWatch 5xx 0 건** — smoke 동안.

### 3.6 회귀 / 회생 가능 시그널
- staging 부하 (예: 5 vu × 30s) 후 ALB 5xx 0 건, BE 평균 latency 200ms 이하 (read endpoint).
- DB CPU 50% 이하 / connection pool 사용률 80% 이하.

---

## 4. P2 — 외부 SaaS 활성화

목표: B17~B22 외부 SaaS 실 impl 통합. ENV swap 만으로 production 활성화 가능.

### 4.1 S3 / FileStorage (R7)
- [ ] `S3FileStorage` impl 작성 (`@ConditionalOnProperty(... havingValue="s3")`).
- [ ] bucket 정책 — versioning + lifecycle (180일 후 Glacier) + 권한 (presigned PUT 만 허용).
- [ ] FE `useUploadFlow` 의 `simulateUpload` 를 실 PUT 으로 교체.
- [ ] concept / record / saenggibu / lessons 전체 회귀 (영상 / 이미지 / PDF link).

### 4.2 PortOne / Payment (R4)
- [ ] R4 mitigation (Mock 조건 부착) 선행.
- [ ] PortOne (또는 Toss) 계정 + secret SSM 등록.
- [ ] `bkurs.external.payment.impl=portone` ENV swap.
- [ ] subscription 생성 → 결제 → refund → webhook 처리 e2e.
- [ ] `PaymentController` webhook secret 검증 + idempotency key 정책.

### 4.3 Notification (SendGrid / SES)
- [ ] sendgrid 키 SSM 등록.
- [ ] `bkurs.external.notification.impl=sendgrid`.
- [ ] 가입 환영 / 결제 영수증 / 미납 알림 mail 흐름 검증.

### 4.4 AI (Claude) / STT (Clova)
- [ ] Anthropic / Clova 키 SSM 등록.
- [ ] rate limit / timeout / fallback 정책 결정.
- [ ] cost monitoring — 월 한도.

### 4.5 PDF (R7)
- [ ] `RealPdfGenerator` impl (openhtmltopdf / wkhtmltopdf / 외부 SaaS).
- [ ] record PDF e2e — 학생 record / 학원장 PDF.

### 4.6 RabbitMQ (선택)
- [ ] AmazonMQ for Rabbit 또는 self-hosted RabbitMQ — production 진입 시점 결정.
- [ ] DLQ 정책 / 알람.

> P2 의 모든 단계는 env / SSM 추가 + ENV swap 으로만 동작 (코드 변경 무 — 이미 gateway 분리).

---

## 5. P3 — production blue/green

### 5.1 인프라
- [ ] VPC + 2 AZ + RDS Multi-AZ 또는 Aurora.
- [ ] ECS service 2 (blue / green) + ALB target group 2 — CodeDeploy 통합.
- [ ] CloudFront + S3 (FE) — 정적 빌드 디플로이 + cache invalidation.
- [ ] Route53 — `bkurs.com` (FE) + `api.bkurs.com` 또는 same-site (`bkurs.com/api/*`) — [[07-open-questions#Q2]].
- [ ] WAF (선택) — 가입 endpoint 의 rate limit + bot 차단.

### 5.2 절차
1. blue (현재 prod) — 살아있음.
2. 새 image push → green ECS service 갱신 → ALB target group 가중 0%.
3. smoke (자동 — P1 §3.5 시나리오를 prod 도메인으로).
4. 가중 swap: 0% → 10% → 50% → 100% (10분 간격 + 5xx / latency 알람 확인).
5. 100% green 안정 후 blue 회수.
6. rollback 기준 (다음 중 하나):
   - 5xx error rate > 1% (1분 평균)
   - p95 latency > 1.5초 (5분 평균)
   - DB error log spike
   - 알람 트리거 시 즉시 weighted routing 0% 으로 복귀 + blue 재활성.

### 5.3 DB 마이그레이션 주의
- Flyway `validate-on-migrate=true` — 운영 마이그레이션 시:
  - 신규 V`n+1` 만 추가. 기존 스크립트 절대 변경 금지.
  - prod 배포 첫 단계는 BE 빈 image 가 마이그레이션 적용 → 이후 트래픽 swap.
  - long-running migration (대용량 alter) 은 사전 PR + dba 협의 ([[07-open-questions#Q5]]).

---

## 6. P4 — 운영 / 회귀 / 관측

- [ ] CloudWatch Logs + structured log (`request_id`, `account_id` 마스킹).
- [ ] X-Ray 또는 OpenTelemetry — distributed tracing.
- [ ] Sentry (BE + FE) — error tracking. `SENTRY_DSN` 이미 env 예시 존재.
- [ ] CloudWatch alarm:
  - 5xx > 1% / 5min
  - DB CPU > 80% / 10min
  - cookie refresh 실패율 > 5%
  - mock SaaS 활성 감지 (R11)
- [ ] Spring Batch:
  - `OverduePaymentAlertJob` 활성화 시점 결정.
  - cron 정책 + 실행 결과 audit.
- [ ] regression suite:
  - BE `./gradlew check` — 전체 통과 필수.
  - FE vitest + Playwright critical-path.
  - smoke (수동 또는 schedule) 의 일일 실행.

---

## 7. 지금 할 일 / 나중에 할 일 — 요약 체크리스트

### 지금 할 일 (P0)
- [ ] R1 signup role whitelist
- [ ] R5 swagger prod 비활성
- [ ] R4 MockPaymentGateway 조건 부착
- [ ] R3 concept 6 파일 cutover
- [ ] R2 Master role 운영 정책 (block / banner)
- [ ] FE 회귀 (vitest critical-path) — 최소
- [ ] smoke 시나리오 문서화

### 나중에 할 일 (P1+)
- [ ] staging 인프라 (VPC / RDS / ECS / ALB / SSM / CloudFront)
- [ ] CI/CD deploy step 활성화 + stg/prod 분리
- [ ] R7 S3 / PDF 실 impl
- [ ] R9 multi-origin CORS / 도메인 정책
- [ ] R10 Flyway baseline 정책 명문화
- [ ] R11 prod startup mock 감지
- [ ] PortOne / SendGrid / Claude / Clova ENV swap
- [ ] blue/green CodeDeploy + rollback runbook
- [ ] CloudWatch + Sentry + X-Ray
- [ ] Spring Batch enable

> 의사결정 필요 항목은 [[07-open-questions]].
