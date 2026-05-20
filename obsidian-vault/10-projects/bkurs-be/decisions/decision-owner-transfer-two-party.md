---
title: "B4D 결정 — 학원장 이관 (양측 confirm + 감사 trail + ACTIVE transfer 1개)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T10:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, owner-transfer, audit, b4d]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-management-owner-policy]]"
  - "[[decision-plan-subscription-model]]"
home_hub: bkurs-be
---

# B4D 결정 — 학원장 이관 (양측 confirm)

## 결론

1. **`academy_owner_transfer` 별도 테이블** — academy_id + from_owner_id + to_owner_id + status + audit timestamps.
2. **양측 confirm flow**: FROM 이 initiate → TO 가 confirm/reject (또는 FROM cancel). 한쪽 일방 변경 금지.
3. **status**: PENDING / CONFIRMED / REJECTED / CANCELLED. 학원당 ACTIVE (PENDING) transfer 1개만.
4. **TO 학원 미보유 검증** — 1인 1학원 정책 유지. TO 가 이미 학원 보유면 confirm 시 409.

## 왜 필요한가

- 학원장 변경 시나리오: 폐업 / 인수합병 / 명의 변경. 운영자 (MASTER) 가 SQL 로 변경하면 감사 부재 + 양측 동의 없음.
- 양측 동의 없으면 도용 위험 (FROM 이 모르는 사이에 인계 / TO 가 모르는 사이에 받음).
- audit trail (transfer row 자체) 로 분쟁 / 회계 / 운영 추적.

## 안 하면 뭐가 문제

- transfer endpoint 없으면 SQL 또는 MASTER 강제 변경 = 운영 위험 + 감사 부재.
- 일방 변경 (FROM 만) 가능하면 TO 가 모르는 학원 떠안음 → 책임 분쟁.
- ACTIVE 1개 제한 없으면 같은 학원에 여러 transfer 진행 → 경합.

## 4 대안 비교 — 이관 흐름

### A. MASTER 직접 변경 — 기각

- 단점: 양측 동의 없음. 운영자 1명이 학원 데이터 통제 = 보안 사고 위험.
- **기각**.

### B. FROM 일방 변경 (TO confirm 없음)

- 단점: TO 가 모르는 학원 떠안음. 책임 분쟁.
- **기각**.

### C. 양측 confirm (FROM initiate → TO confirm) — **채택**

- 장점: 양측 동의. audit. ACTIVE 1개 제한.
- 단점: TO 응답 없으면 무한 대기 (만료 정책 별도 도입 가능).
- **채택 사유**: 보안 + 감사 + 분쟁 회피.

### D. MASTER 가 양측 확인 후 변경 (3자 합의)

- 단점: 운영자 부담 + 자동화 불가.
- **결정**: C 가 표준. MASTER 강제 변경은 비상 endpoint (B5+) 별도.

## 4 대안 비교 — TO 학원 보유 정책

### a. TO 가 이미 학원 보유 → 409 — **B4D 채택**

- 장점: 1인 1학원 정책 (B3A) 일관. 단순.
- 단점: TO 가 본인 학원을 먼저 cancel 해야 받을 수 있음 — 운영 흐름 복잡.

### b. TO 의 기존 학원 자동 cancel + 신규 받음 — 기각

- 단점: 데이터 소실 위험. TO 의 학생 / 강사 / 구독 모두 영향.
- **기각**.

### c. 다중 학원 허용 (1:N owner)

- 단점: B3A 의 1:1 정책 깨짐. schema 변경 필요.
- **기각**: B5+ academy_member 도입 시.

## 테이블 설계

### `academy_owner_transfer`

```sql
CREATE TABLE academy_owner_transfer (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  academy_id      BIGINT NOT NULL,
  from_owner_id   BIGINT NOT NULL,
  to_owner_id     BIGINT NOT NULL,
  status          VARCHAR(32) NOT NULL,         -- PENDING/CONFIRMED/REJECTED/CANCELLED
  initiated_at    DATETIME(6) NOT NULL,
  decided_at      DATETIME(6) NULL,
  decided_by_account_id BIGINT NULL,
  CONSTRAINT fk_transfer_academy FOREIGN KEY (academy_id) REFERENCES academy(id) ON DELETE CASCADE,
  CONSTRAINT fk_transfer_from FOREIGN KEY (from_owner_id) REFERENCES account(id),
  CONSTRAINT fk_transfer_to FOREIGN KEY (to_owner_id) REFERENCES account(id),
  KEY idx_transfer_academy_status (academy_id, status),
  KEY idx_transfer_to_status (to_owner_id, status)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/owner-transfers | FROM OWNER | 본인 학원 + TO email valid OWNER + TO 학원 미보유 + ACTIVE transfer 없음 |
| GET | /api/v1/academies/me/owner-transfers | FROM OWNER | 본인 학원 outgoing |
| POST | /api/v1/academies/me/owner-transfers/{id}/cancel | FROM OWNER | 본인 PENDING transfer |
| GET | /api/v1/owner-transfers/incoming | OWNER | 본인 → PENDING |
| POST | /api/v1/owner-transfers/{id}/confirm | TO OWNER | TO 본인 transfer + 학원 미보유 |
| POST | /api/v1/owner-transfers/{id}/reject | TO OWNER | TO 본인 transfer |

## 회귀 라인

`OwnerTransferApiTest`:
1. FROM transfer initiate (toEmail) → 201 PENDING
2. 존재하지 않는 to email → 404
3. to 가 OWNER 아님 (STUDENT) → 409
4. 이미 ACTIVE transfer → 409
5. TO incoming 조회 → 1 PENDING
6. TO confirm → academy.ownerAccountId 변경 + status CONFIRMED + FROM 학원 없음 + TO 학원 1개
7. TO reject → REJECTED + academy.ownerAccountId 그대로
8. FROM cancel → CANCELLED
9. 다른 OWNER 가 confirm 시도 → 403
10. TO 가 이미 학원 보유 → 409

## 관련

- [[decision-academy-management-owner-policy]] — 1인 1학원 정책 (본 결정의 전제)
- [[decision-plan-subscription-model]] — academy.ownerAccountId 변경 시 subscription/payment 영향 없음 (모두 academy_id 기반)
- 후속: B5+ 학원 인수 시 데이터 검토 (강사/공지/구독 모두 유지)
