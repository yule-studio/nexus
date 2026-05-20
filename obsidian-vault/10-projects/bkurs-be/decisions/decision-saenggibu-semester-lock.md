---
title: bkurs-be — Middle Saenggibu 학기 lock 정책 ADR
kind: decision
status: accepted
created_at: 2026-05-21
tags: [bkurs-be, saenggibu, schema, lock, adr]
related:
  - "[[decision-suhaeng-round-table]]"
  - "[[envelope-migration-progress]]"
  - "[[bkurs-be]]"
home_hub: "[[bkurs-be]]"
---

# bkurs-be — Middle Saenggibu 학기 lock 정책 ADR

## 결정

중등 생기부 (`middle_saenggibu`) 는 **학원 × 학년 × 학기** 단위로 lock 한다.

- `middle_saenggibu_lock` — composite PK `(academy_id, grade, semester)` + deadline_at +
  locked_at.
- 학생 본인 entry 는 `student_account_id × grade × semester × subject` UNIQUE.
- service `ensureNotLocked(academyId, grade, semester)` — locked_at != null 이면 upsert
  거부 → `TermLockedException` → 409 SGB-4091.

## 왜 이 구조 (3 의문 + 답)

### 1. 왜 학생 row 가 아닌 학기 단위 lock?

같은 학기의 다른 과목 entry 끼리 mass-lock 이 필요 — 학원장이 "2학년 1학기 마감" 한 번
누르면 모든 과목 잠금. 학생 row 별 lock 이면 학원장이 수십~수백 행 클릭해야.

### 2. 왜 deadline_at + locked_at 분리?

- `deadline_at` 만 채우면 학생이 "마감 임박" 만 보는 알림 상태.
- `lockNow=true` 명령으로 `locked_at` 채워야 실제 잠금.

이 분리로 학원장이 (1) 사전에 마감 시점 안내 → (2) 실제 마감일에 lock 누르는 2단계 운영.

### 3. 왜 composite PK 사용?

`(academy_id, grade, semester)` 가 자연 키. surrogate id 두면 `findByTerm` 매번 추가
join. JPA `@EmbeddedId` 로 깔끔히 표현 + lock 조회는 PK lookup → O(1).

## 적용 맥락

### lock 설정 흐름

```java
// 학원장이 본인 학원 2학년 1학기 마감 안내
service.setLock(ownerId, academyId, 2, 1,
    OffsetDateTime.parse("2026-12-31T23:59:00+09:00"),
    false);  // deadline 만 — 학생 수정 가능

// 마감일 도래 → 학원장이 잠금
service.setLock(ownerId, academyId, 2, 1, sameDeadline, true);
// 이후 학생 upsert → SGB-4091
```

### lock 해제는 어떻게?

현재 미구현 — 실수 방지 (실수로 푸르고 학생이 또 수정하면 의미 없음). 필요시 별도
`POST /academies/{id}/saenggibu/lock/unlock` 추가 (관리자만, audit log 동반).

### FE 시각화

- 학생 화면 — lock 정보 받아서 `locked` 면 textarea disabled + 안내 문구.
- 학원장 화면 (후속) — 학기 카드에 "마감 D-N" / "잠금 완료" 뱃지.

## 후속 의문

- **고등 생기부 (`student_record`)** 는 별도 entity (V18). 학기 lock 정책 적용은 후속
  cycle — 도메인 분리 유지 (고등은 entry-row, 중등은 학기-row 구조 차이).
- **lock 해제 audit**: 누가 언제 풀었는지 추적 필요해지면 `lock_history` 보조 테이블.

## 관련 노트

- [[decision-suhaeng-round-table]] — 같은 cycle 의 짝 ADR
- [[envelope-migration-progress]] — Saenggibu 도 envelope 표준
- [[bkurs-be]]

## 참고

- Entity: `bkurs-be/src/main/java/com/bkurs/saenggibu/domain/MiddleSaenggibuLock.java`
- Flyway: `bkurs-be/src/main/resources/db/migration/V27__create_middle_saenggibu.sql`
- Service: `SaenggibuService.ensureNotLocked()`
- 테스트: `SaenggibuApiTest.ownerSetLockBlocksStudent` / `ownerSetLockDeadlineOnly`
