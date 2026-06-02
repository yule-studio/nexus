---
project: bkurs
type: architecture-note
status: draft
updated: 2026-05-31
tags: [project, bkurs, fe, be, deployment, risk]
home_hub: projects
related:
  - "[[01-project-map]]"
  - "[[02-fe-flow-inventory]]"
  - "[[03-be-api-contract]]"
  - "[[04-domain-architecture-reading]]"
  - "[[05-operational-risk-register]]"
  - "[[06-deployment-readiness-roadmap]]"
  - "[[07-open-questions]]"
  - "[[../bkurs-fe/bkurs-fe]]"
  - "[[../bkurs-be/bkurs-be]]"
---

# bkurs — 통합 분석 인덱스 (FE + BE)

> 본 폴더는 `bkurs-fe` + `bkurs-be` 두 레포의 **통합 운영 관점** 정리.
> 레포별 hub 는 [[../bkurs-fe/bkurs-fe]] / [[../bkurs-be/bkurs-be]] 가 SSoT.
> 본 문서들은 **운영 배포 준비를 위한 위험 / 우선순위 해석** 에 초점.

분석 대상 (snapshot 일자 2026-05-31):
- FE: `/Users/masterway/masterway-dev/bkurs/bkurs-fe` (Vite 8 + React 19 + TS, src/router.tsx 기준 4 role)
- BE: `/Users/masterway/masterway-dev/bkurs/bkurs-be` (Spring Boot 3.5 + Java 21, 70+ controller)

## 1. 현재 결론 요약 (소프트하게)

| 영역 | 현재 상태 | 위험도 | 비고 |
| --- | --- | --- | --- |
| 인증 / JWT / refresh 회전 | BE + FE interceptor 까지 완비 | 중 | public signup 이 role 검증 없음 — OWNER/MASTER 자기발급 가능 (R1) |
| API envelope | BE `ApiEnvelope<T>` + FE `unwrap` 일관 | 낮음 | logout/204, PDF blob 등 예외 패턴은 컨트롤러별 명시 필요 |
| 도메인 controller | 70+ endpoint, `/api/v1/{role}/me` 패턴 정착 | 낮음 | role 경계 명확. PreAuthorize 활용 검토 필요 |
| FE 데이터 흐름 | 학생/admin 대부분 BE API. master 9개 페이지 중 **8개가 localStorage + mock** | 높음 | master 운영 노출 시 데이터 신뢰성 0 (R2) |
| Supabase 직접 호출 | concept 도메인 (high/middle) + admin 일부 placeholder + 영상 storage placeholder | 높음 | 인증 분리 / row-level secret / 운영 ownership 불명확 (R3) |
| 외부 SaaS (S3 / Payment / Email / AI / STT / PDF / MQ) | 전부 mock default + `@ConditionalOnProperty` 스왑 구조 | 중 | mock 운영 잔류 + MockPaymentGateway 의 conditional 누락 잠재 버그 (R4) |
| Swagger / OpenAPI | `/swagger-ui.html`, `/v3/api-docs` `permitAll` | 중-높 | production 노출 시 API 표면 / 예시 데이터 유출 (R5) |
| DB / Flyway | `ddl-auto=validate` + `validate-on-migrate=true`, V1~V29 | 낮음 | baseline 정책 없음. prod 첫 배포 시 baseline 결정 필요 |
| CORS / cookie | `allowedOriginPatterns` + credentials, prod `secure=true` 강제 | 낮음 | `FRONTEND_URL` 단일 — 다 환경 prod-stg 분리 시 멀티 origin 필요 |
| CI/CD | BE: image build OK, deploy step **주석 처리** / FE: lint+typecheck+build only, deploy 없음 | 중-높 | 실 배포 파이프라인 미완성 (R6) |
| Blue/Green prod | 인프라/실 배포 step 부재 | 중-높 | 후행 작업 |

> "가능 / 불가능" 으로 단정하지 않는다. 위 위험을 **선행조건으로 잡아 정리한 다음** 단계적으로 통과시키는 것이 본 문서들의 목적.

## 2. 다음에 읽을 순서

1. [[01-project-map]] — FE/BE 큰 그림 + role 별 화면 / 도메인 패키지 / 흐름도
2. [[02-fe-flow-inventory]] — role × route × 데이터 소스 매트릭스 + supabase / mock 별도 목록
3. [[03-be-api-contract]] — 도메인별 endpoint 표 + envelope 적용 / 예외 규칙
4. [[04-domain-architecture-reading]] — 도메인 경계 해석 (auth / academy / course / record / concept ...)
5. [[05-operational-risk-register]] — 위험 레지스터 (R1~R10+, Critical / High / Medium / Low)
6. [[06-deployment-readiness-roadmap]] — staging → prod 순서 + smoke / rollback
7. [[07-open-questions]] — 사람 결정 필요한 항목

## 3. Recommended Next 10 Actions (요약 — 자세한 근거는 각 문서)

- [ ] **A1.** Public `signup` 의 role 화이트리스트 적용 (STUDENT_HIGH/STUDENT_MIDDLE 만). OWNER / MASTER / TEACHER 는 별도 invite/bootstrap. ([[05-operational-risk-register#R1]])
- [ ] **A2.** Swagger / OpenAPI 의 production 노출 게이트 (env profile + IP allowlist or 별도 admin auth). ([[05-operational-risk-register#R5]])
- [ ] **A3.** `MockPaymentGateway` 에 `@ConditionalOnProperty(...impl, mock, matchIfMissing=true)` 부착 — 나머지 6 gateway 와 패턴 통일. ([[05-operational-risk-register#R4]])
- [ ] **A4.** Master 페이지의 localStorage / mock-data 종속 인벤토리화 + 운영 노출 우선순위 결정 (운영 release 에서 임시 비활성 / read-only banner). ([[02-fe-flow-inventory]])
- [ ] **A5.** Supabase 직접 호출 (`concept` 도메인 + 영상 storage placeholder) 의 소유권 결정 — `useConceptDiagnosis` 가 이미 BE 계약 보유. cutover 일정 잡기. ([[07-open-questions#Q1]])
- [ ] **A6.** S3 (FileStorageService) 실 구현 + `bkurs.external.file-storage.impl=s3` 환경에서 e2e 업로드 검증 — concept / record / saenggibu / file controller 모두 dependent. ([[06-deployment-readiness-roadmap]])
- [ ] **A7.** `.github/workflows/deploy-dev.yml` 의 commented deploy step 하나 선택 + 자동화 (ECS / SSH / k8s 중 1). FE deploy workflow 신설. ([[05-operational-risk-register#R6]])
- [ ] **A8.** prod cors `FRONTEND_URL` 의 multi-origin 정책 결정 (staging / prod 동시 차단 정책). ([[07-open-questions#Q2]])
- [ ] **A9.** Flyway baseline 정책 명문화 — prod 첫 배포 시 빈 DB / 기존 DB 분기. ([[06-deployment-readiness-roadmap]])
- [ ] **A10.** smoke test 시나리오 정의 (회원가입 → connect → 학원 코드 입력 → roadmap → record 1건 등). ([[06-deployment-readiness-roadmap]])

> 본 인덱스는 결론을 단정하지 않는다. 각 항목은 후속 PR / 결정에서 reject 될 수 있다.
