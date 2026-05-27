---
title: bkurs — 시작하기 (BE + FE 공용 진입점)
kind: guide
status: active
created_at: 2026-05-21
tags: [bkurs, getting-started, onboarding]
related:
  - "[[bkurs-be]]"
  - "[[../bkurs-fe/bkurs-fe]]"
  - "[[external-services-guide]]"
  - "[[env-deployment-guide]]"
  - "[[../bkurs-fe/screen-logic-and-api-map]]"
home_hub: "[[bkurs-be]]"
---

# bkurs — 시작하기

> **여기가 진입점입니다.** bkurs 프로젝트 (BE + FE) 의 모든 문서가 어디에 있는지, 본인의
> 역할에 맞춰 무엇을 어떤 순서로 봐야 하는지 한 곳에서 안내.

## 1. 역할별 5분 진입 가이드

### 🆕 신규 개발자 — "어디서부터 손대지?"
1. **본 노트** — 전체 그림 파악 (5분)
2. **[[env-deployment-guide]] §2** — local 0 부터 띄우기 (BE 8080 + FE 5173)
3. **[[../bkurs-fe/screen-logic-and-api-map]]** — 화면 1개 골라서 "어떤 hook → 어떤
   endpoint" 추적 (10분)
4. **[[../bkurs-be/architecture-api-foundation 같은 화면의 BE controller 코드 한 번
   훑기]]** — 실 API 흐름 체감

### 🚀 dev / prod 배포 담당 — "env / secret 어떻게 챙기지?"
1. **[[env-deployment-guide]] §3** — env 표 (plain / EKS Secret 분류)
2. **[[external-services-guide]]** — 각 SaaS 발급 절차 + 키 매핑
3. **[[env-deployment-guide]] §4** — `company-infra` 팀에 인계할 secret 목록
4. **[[env-deployment-guide]] §5** — CI/CD repo secret + IRSA 권장 사유

### 🏢 외부 인계 (company-infra / 운영자) — "인프라 측에서 챙겨야 할 것"
1. **[[env-deployment-guide]] §3.1** — 큰 그림 (Terraform / EKS / Secret 흐름)
2. **[[env-deployment-guide]] §3.2** — 받아야 하는 prod env 목록 (전수)
3. **[[env-deployment-guide]] §4** — 인계 항목 (image / health / probe / IRSA)
4. **[[env-deployment-guide]] §7** — secret rotation 주기

### 🎨 FE 화면 추가 — "새 화면 어떻게 만들지?"
1. **[[../bkurs-fe/architecture-api-foundation]]** — ApiEnvelope 구조
2. **[[../bkurs-fe/decisions/decision-hook-convention|hook 컨벤션 ADR]]** — `_hooks/`
   금지, `lib/hooks/` 만
3. **[[../bkurs-fe/screen-logic-and-api-map]] §7** — 신규 화면 체크리스트
4. **[[fe-domain-clients-complete]]** — 18 도메인 client 매핑표

### 🔧 BE 도메인 추가 — "새 API 어떻게 추가?"
1. **[[envelope-migration-progress]] §3.1** — 도메인 작업 template (Errcode/Handler/Controller/Test)
2. **[[envelope-migration-progress]] §3.4** — 전수 점검 grep 명령
3. **[[external-services-guide]]** — 외부 SaaS 추가 시 `bkurs.external.*` prefix +
   `@ConditionalOnProperty` Mock fallback

---

## 2. 문서 지도 (정리된 노트)

### 2.1. 진입 가이드 (본 폴더 = bkurs-be)
- **[[getting-started]]** ← 본 노트 (전체 진입점)
- **[[external-services-guide]]** — 외부 SaaS 7+ 종 발급/설정/fallback
- **[[env-deployment-guide]]** — env 표 + 배포 흐름 + company-infra 경계

### 2.2. BE 아키텍처 / 진척
- **[[bkurs-be]]** — 프로젝트 hub (PR 히스토리 + 결정 누적)
- **[[architecture-async-mq-batch]]** — 비동기 / RabbitMQ / Spring Batch
- **[[architecture-hybrid-external-services]]** — gateway 추상화 + Mock swap 패턴
- **[[envelope-migration-progress]]** — 19 도메인 envelope 표준화 진척표 + template
- **[[external-services-accounts]]** — 계정 발급 진척 추적 (체크리스트)
- **[[cycle-3-retrospective]]** / **[[cycle-5-envelope-completion]]** — 사이클별 회고

### 2.3. BE 결정 (decisions)
- `decisions/decision-account-single-table-role-status.md` — 단일 account 테이블 + role
- `decisions/decision-auth-architecture-jwt-refresh.md` — JWT + refresh cookie
- `decisions/decision-suhaeng-round-table.md` — round-table state machine
- `decisions/decision-saenggibu-semester-lock.md` — 학기 lock composite PK
- … `decisions/` 폴더 전체

### 2.4. FE 노트 (bkurs-fe 폴더)
- **[[../bkurs-fe/bkurs-fe]]** — FE 프로젝트 hub
- **[[../bkurs-fe/screen-logic-and-api-map]]** — 화면별 hook + endpoint 매핑표 (status DONE/MOCK/N/A)
- **[[../bkurs-fe/architecture-api-foundation]]** — ApiEnvelope unwrap + apiClient
- **[[../bkurs-fe/fe-domain-clients-complete]]** — 18 도메인 client 목록
- **[[../bkurs-fe/decisions/decision-hook-convention]]** — `lib/hooks/` 2-tier
- **[[../bkurs-fe/retrospectives]]** — FE 사이클 회고

---

## 3. 현재 상태 한 줄 요약 (2026-05-21)

- **BE**: 19 도메인 + auth + 입시(past_question) — 100+ endpoint, envelope 통일.
  외부 SaaS gateway 5종 (S3/Email/AI/STT/Payment) + Mock fallback.
- **FE**: 18 도메인 client + 150+ endpoint + 130+ hook. 화면 swap **25개 DONE**
  (admin 9 / 학생 15 / master 1). 입시 / master 본사 9 화면이 남음.
- **인증 흐름** 4 그룹 (학생 high/middle / admin / master) 모두 실 BE 로그인 동작.

자세한 swap 현황은 [[../bkurs-fe/screen-logic-and-api-map]] 참조.

## 4. "이것만은 알아야" — 함정 / 컨벤션

- **secret 절대 금지 commit**: `.env.local` (gitignore), AWS key, ANTHROPIC_API_KEY 등.
  실수 시 [[env-deployment-guide]] §7 의 incident response 참조.
- **prod 는 mock 금지**: 외부 SaaS env 가 비면 자동 Mock 활성화 — local/dev 만 OK.
  prod 는 [[external-services-guide]] §3 환경별 매트릭스의 "실 API" 칸을 모두 채워야 함.
- **회원가입 grade 1~6**: 1~3=중1~중3, 4~6=고1~고3 ([[../bkurs-fe/screen-logic-and-api-map]] 참조).
- **company-infra 경계**: Terraform / EKS Secret 자체는 별도 레포. 본 가이드는
  bkurs-be / bkurs-fe 가 *받는* env 만 다룸.

## 5. 다음에 볼 것

- 신규 합류자 → [[env-deployment-guide]] §2 (local 띄우기)
- 화면 작업 → [[../bkurs-fe/screen-logic-and-api-map]]
- API 추가 → [[envelope-migration-progress]] §3.1 template
- 외부 SaaS 추가 → [[external-services-guide]] §3 + [[env-deployment-guide]] §6 체크리스트

---

> **문서 누락 / 오류 발견 시**: 본 노트의 §2 지도를 갱신하거나, 새 가이드를 추가한 뒤
> 본 노트 §2 와 해당 노트의 "관련 노트" cross-link 를 양쪽 모두 보강.
