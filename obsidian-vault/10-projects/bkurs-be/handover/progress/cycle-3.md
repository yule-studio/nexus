---
title: bkurs-be — Cycle 3 회고 (Suhaeng + Saenggibu + FE swap)
kind: retrospective
status: closed
created_at: 2026-05-21
tags: [bkurs-be, bkurs-fe, retrospective, cycle-3]
related:
  - "[[decision-suhaeng-round-table]]"
  - "[[decision-saenggibu-semester-lock]]"
  - "[[envelope-migration]]"
  - "[[../../bkurs-fe/architecture-api-foundation]]"
home_hub: "[[../bkurs-be]]"
---

# bkurs-be — Cycle 3 회고

## 1. 핵심 요약

tech-lead plan 의 Cycle 3 (BE 신규 도메인 + FE swap) 5 PR 모두 머지.

| PR | 설명 | 결과 |
|---|---|---|
| bkurs-be #107 | PR-7 Suhaeng entity + V26 + state machine guard | 8 entity test PASS |
| bkurs-be #109 | PR-8 Suhaeng controller × 3 + service + MQ 발행 | 8 integration PASS |
| bkurs-be #111 | PR-9 Saenggibu (Record) entity + Lock + 3 controller | 7 integration PASS |
| bkurs-fe #67 | PR-10 suhaeng/saenggibu api + hook + 2 화면 | typecheck PASS |
| vault | PR-11 ADR × 2 + 본 회고 | 본 노트 |

총 추가:
- BE entity 4 (Suhaeng / SuhaengRound / MiddleSaenggibu / Lock).
- BE Flyway 2 (V26 / V27).
- BE controller 6 (Student/Teacher/Owner × 2 도메인).
- BE endpoint 14.
- FE api client 2 + react-query hook 13 + 화면 2.
- ADR 2.

## 2. 잘된 점

### envelope template 의 위력
phase 1 (#101/#103/#105) 에서 잡은 3 commit 분할 (Enum+Handler / Controller / Test +
helper) 을 Cycle 3 신규 도메인에도 그대로 적용. 새 도메인 4개 모두 같은 형태 →
검토 부담 최소 + 일관성.

### round-table 결정의 효과
사용자가 round-table 선택 한 덕분에 inroad-high 의 "v1/v2 흐름 표시" UI 가 자연스럽게
가능. single-table 였으면 audit log + join 필요했음.

### Hook 컨벤션 ADR 의 효과
Cycle 1 에서 page-local vs lib/hooks 결정 → Cycle 3 의 useSuhaeng / useSaenggibu 가
어디 가야 하는지 결정 시간 0초. 두 번째 cross-role hook (useConceptDiagnosis 다음)
이라 패턴 입증.

### Spring Batch test 환경변수 패턴
`DB_PASSWORD='...' ./gradlew test --tests "..."` 이 매 cycle 통과 — 환경 설정 표준화.

## 3. 어려웠던 점

### Membership repository API 불일치
SaenggibuService 초안에서 `findFirstByAccountIdAndStatusOrderByJoinedAtDesc` 호출 → 실제는
`findFirstByAccountIdAndAcademyIdAndStatusIn` 만 존재. compile fail 발견 → service 수정.
교훈: 신규 도메인이 다른 도메인 repository 호출할 때는 시그니처 grep 먼저.

### Composite PK 의 첫 사용
`MiddleSaenggibuLockId` @Embeddable 첫 사용. `findByTerm` 헬퍼로 사용성 보완 — JPA 의
ID 객체 직접 노출은 피함.

### 화면 단순화 트레이드오프
HighStudentSuhaeng / MiddleStudentRecord 화면은 BE 도달 전 작성 → 시각 디테일 (theme,
animation, mobile 최적화) 보류. BE smoke 후 후속 cycle 에서 다듬기.

## 4. 후속 작업 (자연스러운 다음 cycle)

### Cycle 4 — concept S3 화면 swap
- BE dev 환경 부팅 필요.
- concept 화면 multi-row vs single-row 모델 결정 필요.
- 보류 — 사용자 결정 + BE 부팅 후.

### Cycle 5+ — envelope phase 2-5
- 도메인 묶음 4 PR (course/homework/reading → question/interview/simulation →
  record/performance/statement → plan/payment/search/announcement/discussion).
- phase 1 + Cycle 3 신규 = 6 도메인 envelope. 잔여 ~55 controller.

### B43 — AI 분석 consumer
- MQ `AI_ANALYZE` 발행은 Cycle 3 에서 시작. consumer 가 실제 AI 호출 + round 갱신은
  별도 PR. WebSocket / polling 정책 같이 결정.

### B2 잔여 화면
- admin/SuhaengManage (학원장이 학생에게 일괄 부여)
- master/SuhaengApproval (운영자가 학원 lock 정책 관리)

## 5. 통계

- Cycle 3 5 PR 머지 — Cycle 2 종료부터 1 세션.
- 추가 코드: BE +2300 lines / FE +680 lines / vault +400 lines.
- 신규 통합 test: 22 case (8+7+7), 모두 PASS.
- 회귀: 0.

## 관련 노트

- [[decision-suhaeng-round-table]]
- [[decision-saenggibu-semester-lock]]
- [[envelope-migration]] — phase 1 종료, Cycle 5+ 진행 예정
- [[../../bkurs-fe/architecture-api-foundation]] — FE 쪽 S1/S2 + Cycle 3 hook 추가
- [[../../bkurs-fe/decisions/decision-hook-convention]] — Cycle 3 의 hook 위치 결정 근거

## 참고

- BE entity 위치: `bkurs-be/src/main/java/com/bkurs/{suhaeng,saenggibu}/`
- FE api / hook: `bkurs-fe/src/lib/{api,hooks}/`
- FE 화면: `bkurs-fe/src/pages/{high-student/_pages/suhaeng,middle-student/_pages/record}/`
