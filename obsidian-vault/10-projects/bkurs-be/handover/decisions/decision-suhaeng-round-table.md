---
title: bkurs-be — Suhaeng round-table 분리 ADR
kind: decision
status: accepted
created_at: 2026-05-21
tags: [bkurs-be, suhaeng, schema, state-machine, adr]
related:
  - "[[envelope-migration]]"
  - "[[async-mq-batch]]"
  - "[[bkurs-be]]"
home_hub: "[[bkurs-be]]"
---

# bkurs-be — Suhaeng round-table 분리 ADR

## 결정

수행평가 도메인을 **round-table 분리** 구조로 둔다.

- `suhaeng` — 학원장이 학생에게 부여한 **과제 메타** (1 학생 1과제 1행). subject /
  title / deadline_at / weight 만.
- `suhaeng_round` — 매 제출/피드백 **round** (suhaeng_id + round_no UNIQUE).
  round_no + status + student_answer + ai_analysis_json + teacher_feedback +
  final_score.

current status of Suhaeng = `latest round` (round_no 가장 큰 row) 의 status.

## 왜 이렇게 결정했나

사용자가 3 옵션 중 round-table 분리를 선택 (2026-05-21). 평가:

| 옵션 | 장점 | 단점 | 평가 |
|------|------|------|------|
| **A. Single-table + status enum** | 구조 단순, query 1번 | 재제출 history 사라짐 — 학생 답안 v1/v2 추적 X. 강사 피드백도 last-write-wins | 학습 분석에 부적합 |
| **B. Round-table 분리 (채택)** | round 별 답안/피드백/AI 분석 완전 보존. 학생 본인이 본인 성장 시각화 가능 | query 약간 복잡 (latest round 조회 필요) | **균형 우수** |
| **C. Single-table + audit log 분리** | main 단순 + audit 투명 | audit log 가 read 경로의 1급 시민이 되지 못해 화면 추적이 추가 join | 운영 투명성만 좋음 |

선택 B 의 결정적 이유:
- inroad-high 의 화면이 **round 별 답안/피드백 명시 표시** (학생이 "내가 v1 에서 무엇을
  썼고 강사가 무엇을 지적했고 v2 에서 무엇을 고쳤는지" 흐름 따라가는 UI). audit log
  로는 화면 못 그림.
- AI 분석 결과는 round 별 — v1 의 AI 분석 vs v2 의 AI 분석 비교가 학습 가치.
- 강사도 "이 학생이 처음에 이렇게 썼는데 피드백 받고 이렇게 바뀜" 을 한 화면에서 보여줄
  필요.

## state machine

```
(신규)           → SUBMITTED         — 학생 답안 제출 (round_no=1)
SUBMITTED        → AI_ANALYZED       — async AI 분석 완료
AI_ANALYZED      → TEACHER_FEEDBACK  — 강사 피드백 작성
TEACHER_FEEDBACK → (round_no+1 신규) → SUBMITTED  — 학생 재제출
AI_ANALYZED      → FINALIZED         — 강사 피드백 생략 후 즉시 확정
TEACHER_FEEDBACK → FINALIZED         — 강사 최종 확정 (final_score 입력)
```

가드:
- `SuhaengRound.recordAiAnalysis()` — SUBMITTED 일 때만.
- `SuhaengRound.recordTeacherFeedback()` — SUBMITTED 또는 AI_ANALYZED 일 때만.
- `SuhaengRound.finalize()` — FINALIZED 가 아닐 때.
- `SuhaengService.resubmit()` — latest round 가 TEACHER_FEEDBACK 일 때만 (1 단계 강제).
- 위반 시 `IllegalStateException` → handler 가 409 SUH-4090 envelope.

## 적용 맥락

### 신규 round 추가 절차 (어플리케이션 코드)

```java
// 첫 제출
SuhaengRound r1 = SuhaengRound.builder()
    .suhaengId(s.getId()).roundNo(1).studentAnswer(answer).build();
roundRepository.save(r1);

// 재제출 — latest round 의 status 가 TEACHER_FEEDBACK 일 때만
SuhaengRound latest = roundRepository.findFirstBySuhaengIdOrderByRoundNoDesc(s.getId())
    .orElseThrow(...);
if (latest.getStatus() != SuhaengStatus.TEACHER_FEEDBACK) throw new IllegalStateException(...);
SuhaengRound r2 = SuhaengRound.builder()
    .suhaengId(s.getId()).roundNo(latest.getRoundNo() + 1).studentAnswer(newAnswer).build();
```

### 인덱스

- `suhaeng_round(suhaeng_id, round_no DESC)` — latest round 조회 O(1).
- 화면이 모든 round 표시할 때는 `findBySuhaengIdOrderByRoundNoAsc` (idx scan).

### 영향

- FE: `SuhaengView` 가 `rounds: SuhaengRoundView[]` 포함 — 한 query 로 전체 흐름 표시 가능.
- AI consumer (B43 후속): MQ `AI_ANALYZE` payload 의 `roundId` 로 정확한 round 갱신.

## 후속 의문

- **재제출 횟수 제한**: 현재 무제한. 학원 단위 정책 (예: round_no <= 3) 필요 시
  service 에 가드 추가.
- **AI 분석 실패 시 retry**: consumer 가 SUBMITTED 에서 멈춤. dead-letter / retry 정책
  은 B41 (WebSocket / progress polling) 과 같이 설계.

## 관련 노트

- [[envelope-migration]] — phase 1 종료, 신규 도메인부터 envelope
- [[async-mq-batch]] — RabbitMQ `AI_ANALYZE` 토픽 위치
- [[decision-saenggibu-semester-lock|Saenggibu 학기 lock ADR]]

## 참고

- Entity: `bkurs-be/src/main/java/com/bkurs/suhaeng/domain/{Suhaeng,SuhaengRound}.java`
- Flyway: `bkurs-be/src/main/resources/db/migration/V26__create_suhaeng.sql`
- Service: `bkurs-be/src/main/java/com/bkurs/suhaeng/service/SuhaengService.java`
- 테스트: `SuhaengEntityTest` (state machine guard) + `SuhaengApiTest` (8 통합)
