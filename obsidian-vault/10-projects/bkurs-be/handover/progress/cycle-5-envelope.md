---
title: bkurs-be — Cycle 5 envelope 마이그레이션 완료 회고
kind: retrospective
status: closed
created_at: 2026-05-21
tags: [bkurs-be, retrospective, envelope, cycle-5]
related:
  - "[[envelope-migration]]"
  - "[[cycle-3]]"
  - "[[../../bkurs-fe/architecture-api-foundation]]"
home_hub: "[[../bkurs-be]]"
---

# bkurs-be — Cycle 5 envelope 마이그레이션 완료 회고

## 1. 핵심 요약

tech-lead plan 의 phase 2 envelope 마이그레이션 — 12 PR 모두 머지. 총 17 도메인
envelope 표준 적용 완료.

## 2. 잘된 점

### template 효과
phase 1 (auth/student-profile/dashboard) 에서 잡은 3 commit 분할 (Enum + Handler /
Controller / Test data() helper) 을 phase 2 의 12 도메인에 그대로 복제. 검토 부담
최소 + 일관성. 각 PR 약 10~15분.

### error code 일관 prefix
도메인별 prefix (AUTH, STU, SUH, SGB, READ, HW, CRS, Q, IVW, SIM, REC, PE, STM, PAY,
SRC, ANN, DSC) — 17 prefix. 4-자리 status seq 의 표준 (4040 not found, 4030 forbidden,
4090 conflict 등) 도 일관.

### cross-domain test 자동 fallback
StudentRecordBridgeApiTest + SearchApiTest 가 envelope 적용된 도메인 + 미적용 도메인을
같이 호출 — `extractId()` helper 가 `data.id` / `id` 양쪽 fallback 으로 호환. 도메인
점진 마이그레이션 동안 회귀 발생 0.

## 3. 어려웠던 점

### Write 도구 — Read 없이 Write 시 fail
신규 enum 파일 옆 기존 handler 파일 갱신 시 일부 turn 에서 Write 가 Read 안 됐다는
이유로 fail. mechanical retry 로 해결했지만 cycle 별 1-2 회 발생.

### test 헬퍼의 envelope unwrap 차이
- 단일 도메인 test: `data()` helper 한 곳 추가 + `body.get()` → `data().get()` mechanical.
- list 응답: `ResponseEntity<List>` → `ResponseEntity<Map>` 로 받아 `body.get("data")` 추출.
- bridge/search test: 도메인 cross-call → `extractId()` fallback 패턴 필요.

세 가지 변형을 phase 별로 한 번씩 해결한 후 모든 도메인에 같은 패턴 복제.

### handler 의 IllegalStateException 처리 위치
statement / suhaeng 두 도메인 같은 IllegalStateException → 도메인별 envelope.fail
매핑. 두 도메인 동시 ExceptionHandler 충돌 X (basePackages 분리).

## 4. 회귀 / 위험 통계

- 신규 통합 test 갱신 (전체 발견): 119 test (10 × 12 phase 2 도메인 + 일부 helpers)
- 회귀 0
- compile fail 0 (모두 첫 commit 후 ./gradlew compileJava PASS)
- Spring context 부팅 실패 0 (DB_PASSWORD env 있을 때)

## 5. 다음 작업 후보

### 즉시 가능 — bkurs-fe envelope unwrap 확장
모든 도메인이 envelope → bkurs-fe 의 api/ 디렉터리에서 새 도메인 client 작성 가능.
suhaeng/saenggibu 외 모든 도메인 client 추가는 사용자 결정 후.

### Cycle 4 — concept S3 화면 swap
사용자 결정 (model 정책) + BE dev 부팅 후. envelope 적용 완료로 화면 swap 의 1 차
제약은 해소.

### Cycle 6 — B43 AI 분석 consumer
suhaeng submit → MQ AI_ANALYZE 발행 (Cycle 3 에서 시작). consumer 가 실제 AI 호출 +
round.recordAiAnalysis() 갱신은 별도 PR. 사용자 의도 (외부 SaaS 실 연동 후) 까지 보류.

### envelope 후속 cleanup
- GlobalExceptionHandler 의 fallback 처리 — 도메인 advice 가 모두 envelope 이면
  GlobalExceptionHandler 도 envelope (이미 적용 — 회귀 0).
- OpenAPI 스펙 갱신 — 응답 schema 가 모두 ApiEnvelope<T> 로 변함. swagger UI 에서
  확인 후 docs 보강.

## 6. 통계

- Cycle 2 (phase 1) — 3 PR (auth/student-profile/dashboard) → 3 도메인
- Cycle 3 (신규) — 5 PR (suhaeng entity/controller, saenggibu, FE swap, vault) → 2 도메인
- Cycle 5 (phase 2) — 12 PR → 12 도메인 (reading/homework/course/question/interview/
  simulation/record/performance/statement/plan/search/announcement/discussion)
- 총 envelope PR: 17 (Cycle 2 + 3 + 5 합산 — 5+12)
- 총 추가 코드: ~3000 lines (controller + handler + enum + test)
- 총 통합 test 갱신: 119 case

## 관련 노트

- [[envelope-migration]] — 진척표 (Cycle 5 마감 갱신)
- [[cycle-3]] — Cycle 3 회고
- [[../../bkurs-fe/architecture-api-foundation]] — FE 쪽 envelope unwrap 구조

## 참고

- 코드 SSoT: `bkurs-be/src/main/java/com/bkurs/common/api/{ApiEnvelope,ApiEnvelopes,ApiMeta}.java`
- 도메인 ErrorCode 17개: `bkurs-be/src/main/java/com/bkurs/<domain>/enums/<Domain>ErrorCode.java`
- 후속: Cycle 4 (concept S3) / Cycle 6 (AI consumer) 또는 OpenAPI / GlobalExceptionHandler 정리.
