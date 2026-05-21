---
title: "B9 결정 — StudentRecord auto-bridge (4 도메인 → 생기부)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T17:25:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, record-bridge, b9]
related:
  - "[[../bkurs-be]]"
  - "[[decision-student-record-categories]]"
  - "[[decision-performance-evaluation-model]]"
  - "[[decision-discussion-model]]"
  - "[[decision-simulation-model]]"
  - "[[decision-reading-note-model]]"
home_hub: bkurs-be
---

# B9 결정 — StudentRecord auto-bridge

## 결론

1. **`student_record_entry` ALTER ADD source_type + source_id + uk(source_type, source_id)** — 같은 source 면 update.
2. **4 source_type**: PERFORMANCE / DISCUSSION / SIMULATION / READING.
3. **4 매핑**: SUBJECT / AUTONOMY / CAREER / READING.
4. **Spring ApplicationEvent same-TX** — 채점 실패 시 record 도 함께 롤백.
5. **author = grader** — record audit 책임 명확.

## 왜 필요한가

- 강사가 채점/평가 후 별도로 생기부에 옮겨 적는 부담 제거.
- 학생부 자료가 데이터 소스 (PE/Disc/Sim/Reading) 와 자동 동기화.
- 강사 수동 record 입력 (source_type=NULL) 은 보존.

## 4 대안 — 동기화 방식

### A. Spring ApplicationEvent same-TX — **채택**

- 장점: 코드 단순, 채점/평가 와 record 가 같은 TX → 부분 실패 없음.
- **채택**.

### B. @TransactionalEventListener(AFTER_COMMIT) async

- 장점: 채점 성공 후 bridge 실패해도 grade 유지.
- 단점: bridge 실패 시 record 누락 — 별도 retry / monitor 필요.
- **기각** (PoC): 후속 production hardening 시 검토.

### C. 명시적 endpoint (POST /promote-to-record)

- 장점: 강사 의도 명확.
- 단점: 매 채점마다 두 번 호출 — 자동화 가치 잃음.
- **기각**.

### D. DB trigger

- **기각**: SQL trigger 는 enum 매핑 / event 정보 한계.

## 4 대안 — idempotency

### A. uk(source_type, source_id) — **채택**

- 장점: DB 레벨 보장. createOrUpdateFromSource pattern.

### B. application-level 검증 + 단순 insert

- **기각**: race condition 가능.

## 매핑 결정

| source | category | 이유 |
|---|---|---|
| PERFORMANCE | SUBJECT | PE = 공식 수행평가 = 교과세특 |
| DISCUSSION | AUTONOMY | 토론 = 자율 활동 |
| SIMULATION | CAREER | 모의지원 = 진로 활동 |
| READING | READING | 독서 = 독서 |

## 회귀 10

1. PE 채점 → SUBJECT entry auto-created
2. PE 재채점 → SUBJECT entry update (same source_id)
3. Discussion 평가 → AUTONOMY entry
4. Simulation review → CAREER entry
5. Reading feedback → READING entry
6. 같은 학생 다른 source → 별도 entry
7. record.author = grader 검증
8. record.content 에 source title + score + feedback 포함
9. 수동 record (source_type=NULL) 은 영향 없음
10. PE DELETE → record 보존 (학생부 영구)

## 관련

- [[decision-student-record-categories]] — 8 카테고리 정의
- 후속: AFTER_COMMIT async / period 자동 결정 (학기 표시) / 통계
