---
title: "B9 task-log — StudentRecord auto-bridge"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T19:45:00+09:00
agent: engineering-agent/coding
tags: [task-log, record-bridge, b9]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-record-bridge]]"
home_hub: bkurs-be
---

# B9 task-log — StudentRecord auto-bridge

## 결과

- 이슈: #51
- PR: #52 (draft, stacked on #50 B7I)
- 브랜치: `feature/issue-51-b9-record-bridge`
- 5 commit (S1~S4 + CHANGELOG fixup)
- 10 통합 회귀 PASS — 4 도메인 통합 검증

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| 20b30de | V23 ALTER + RecordSourceType + entity / repo 보강 | +52 | uk(source_type, source_id) |
| 8ad135a | 4 event + StudentRecordBridgeService | +180 | @EventListener same-TX |
| 8561a27 | 4 service ApplicationEventPublisher + publish | +30 | gradle/evaluate/review/feedback |
| aa5d6dc | StudentRecordBridgeApiTest 10 | +323 | 모든 회귀 PASS |
| 0b5e454 | CHANGELOG B9 (직전 누락 fixup) | +14 | Edit 도구 실패 보상 |

## 매핑

| source | category | trigger |
|---|---|---|
| PERFORMANCE | SUBJECT | PerformanceEvaluationService.gradeByTeacher |
| DISCUSSION | AUTONOMY | DiscussionService.evaluateByTeacher |
| SIMULATION | CAREER | SimulationService.reviewByTeacher |
| READING | READING | ReadingService.submitFeedbackByTeacher |

## 10 회귀 (StudentRecordBridgeApiTest)

1. PE 채점 → SUBJECT record auto-created ✅
2. PE 재채점 → 같은 record update ✅
3. Discussion 평가 → AUTONOMY record ✅
4. Simulation review → CAREER record ✅
5. Reading feedback → READING record ✅
6. 4종 source → 별도 record 4개 ✅
7. PE 재제출 (score null) → bridge 스킵 ✅
8. 수동 record (source_type=NULL) coexist ✅
9. Discussion DELETE topic → record 보존 (학생부 영구) ✅
10. 학생 본인 record 조회 → bridge 4종 모두 노출 ✅

## 선택 이유 — handover

### A. Spring ApplicationEvent same-TX

- **선택 이유**: 코드 단순, 채점/평가 와 record 가 같은 TX → 부분 실패 없음.
- **기각된 대안**: @TransactionalEventListener(AFTER_COMMIT) async — bridge 실패 시 monitor/retry 필요.
- **trade-off**: bridge 실패 시 grade 도 롤백 — PoC OK, production 은 후속 검토.

### B. uk(source_type, source_id) DB 레벨 idempotency

- **선택 이유**: 재채점 시 application-level race condition 회피.
- **기각된 대안**: app-level findBy+save — race 가능.

### C. 매핑 (PERFORMANCE→SUBJECT 등) 고정

- **선택 이유**: 도메인 의미 직결 (PE = 공식 평가 = 교과세특).
- **기각된 대안**: per-instance category 선택 (강사가 입력 시마다 결정) — 자동화 가치 잃음.
- **future**: 강사가 매핑 override 필요하면 별도 endpoint (PATCH /records/{id} 의 category 수정 — 이미 지원).

### D. score null 시 bridge 스킵 (PE/Sim 재제출 케이스)

- **선택 이유**: 재제출 = 채점 무효화 → record 도 무효 / 갱신 보류.
- **기각된 대안**: record 도 score null 로 update — 학생부 노출 가치 떨어짐.
- **trade-off**: 강사가 재채점할 때까지 record 가 이전 점수로 유지 — 명시적 stale.

### E. domain DELETE 시 record 보존 (FK 연결 안 함)

- **선택 이유**: 학생부 영구성 — PE/Topic 삭제는 운영 정리, record 는 학생 평생.
- **기각된 대안**: ON DELETE CASCADE — 학생부 데이터 손실 위험.
- **trade-off**: source_id 가 stale 가능 — 강사 수동 record (overwriteFromSource X) 와 동일하게 보존.

## 관련

- [[../decisions/decision-record-bridge]]
- [[task-log-b7e-student-record]] / [[task-log-b7f-discussion]] / [[task-log-b7g-reading-note]] / [[task-log-b7h-performance-evaluation]] / [[task-log-b7i-simulation]] — 5 source 도메인
- 후속: AFTER_COMMIT async / period 자동 결정 (학기) / B8 PDF export
