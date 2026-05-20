---
title: "B3C 결정 — 학원 공지 + read 추적 (별도 read 테이블 + 단일 학원 가정)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T08:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, announcement, read-tracking, b3c]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-management-owner-policy]]"
  - "[[decision-teacher-invite-assignment]]"
  - "[[decision-academy-connect-membership-flow]]"
home_hub: bkurs-be
---

# B3C 결정 — 학원 공지 + read 추적

## 결론

1. **공지 본문 (`announcement`) + read 상태 (`announcement_read`) 별도 테이블** — 공지는 N readers, 1 row 가 N relationships.
2. **작성 권한: OWNER 만** — B3C 정책. TEACHER 작성 권한은 B4+ 에서 결정 (강사 권한 매트릭스 별도 결정).
3. **read 상태**: 학생/강사가 명시적으로 `POST /read` 호출 시 row insert (upsert). 자동 추적은 안 함.
4. **본인 학원 결정 (`AcademyMembershipResolver`)**: 학원장 = `owner_account_id` / 학생 = ACTIVE membership / 강사 = ACTIVE assignment. 학원 미가입은 빈 응답.

## 왜 필요한가

- 학원장이 공지 / 알림 / 학사 일정 등 학원 운영 메시지를 학생·강사 전체에 게시할 통로 필요.
- read 추적: 학원장이 "공지를 본 학생 수" 파악 → 후속 조치 (개별 연락 / 재공지) 가능.
- 학원장 / 학생 / 강사 모두 다른 권한 — 본인 학원 외 공지 접근 못 함.

## 안 하면 뭐가 문제

- 공지 / 알림 endpoint 없으면 학원장이 외부 도구 (카톡 / 문자) 로 우회 → 학원 운영 통합 끊김.
- read 추적 없으면 "이 공지를 본 사람?" 운영 결정 못 함.
- 권한 분기 없으면 다른 학원 공지가 노출 → 보안 사고.

## 4 대안 비교

### A. announcement 컬럼에 read_account_ids JSON — 기각

- 단점: array growth → 한 row 비대화. JSON 인덱스 부재로 "이 공지 본 사람?" query 느림.
- **기각 사유**: 정규화 위반 + 운영 query 성능.

### B. 별도 `announcement_read` 테이블 — **채택**

- 장점: 정규화. (announcement_id, account_id) 인덱스로 양방향 query.
- 단점: 공지 1 + read N rows 추가 — 운영 비용. 학원 100명 × 공지 100 = 10k row/학원.
- **채택 사유**: 정규화 + query 효율 + 향후 read_at timestamp / device info 등 컬럼 추가 가능.

### C. announcement.read_count + 별도 N rows — 미래 후보

- 장점: read_count 만 빠르게 조회 가능 (drill-down 시 별도 query).
- 단점: 동기화 비용 (insert read row + count update). 일관성 위험.
- **결정**: count 가 hot path 가 되면 별도 PR 도입. B3C 는 단순 정규화 우선.

### D. read 자동 추적 (GET 시 read 마킹)

- 단점: GET 의 부수효과 — RESTful 원칙 위반 + 캐시 안전성 깨짐.
- **기각 사유**: REST 원칙.

## 4 대안 비교 — 작성 권한

### a. OWNER 만 — **B3C 채택**

- 장점: MVP 정책. 강사 권한 매트릭스 미정 상황에서 가장 안전한 default.
- 단점: 강사 작성 흐름 없음 — 운영 피드백 후 풀 가능성 고려.

### b. OWNER + TEACHER

- 단점: 강사 권한 매트릭스 (작성/삭제/PATCH 각각) 결정 필요. B3B 강사 가입은 끝났지만 권한 정책은 별도.
- **결정**: B4+ 의 강사 권한 매트릭스 PR 후 확장.

### c. announcement_role (작성자 role 컬럼) + 화이트리스트 정책 테이블

- 단점: over-design. B3C 범위 초과.
- **기각**.

## 테이블 설계

### `announcement`

```sql
CREATE TABLE announcement (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  academy_id   BIGINT NOT NULL,
  author_id    BIGINT NOT NULL,
  title        VARCHAR(200) NOT NULL,
  body         TEXT NOT NULL,
  created_at   DATETIME(6) NOT NULL,
  updated_at   DATETIME(6) NOT NULL,
  CONSTRAINT fk_announcement_academy FOREIGN KEY (academy_id) REFERENCES academy(id) ON DELETE CASCADE,
  CONSTRAINT fk_announcement_author FOREIGN KEY (author_id) REFERENCES account(id),
  KEY idx_announcement_academy_created (academy_id, created_at DESC)
);
```

### `announcement_read`

```sql
CREATE TABLE announcement_read (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  announcement_id BIGINT NOT NULL,
  account_id      BIGINT NOT NULL,
  read_at         DATETIME(6) NOT NULL,
  CONSTRAINT fk_read_announcement FOREIGN KEY (announcement_id) REFERENCES announcement(id) ON DELETE CASCADE,
  CONSTRAINT fk_read_account FOREIGN KEY (account_id) REFERENCES account(id),
  CONSTRAINT uk_read UNIQUE (announcement_id, account_id),       -- upsert key
  KEY idx_read_account (account_id)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/announcements | OWNER | 본인 학원 |
| GET | /api/v1/academies/me/announcements | OWNER | 본인 학원 |
| PATCH | /api/v1/academies/me/announcements/{id} | OWNER | 본인 학원 공지만 |
| DELETE | /api/v1/academies/me/announcements/{id} | OWNER | 본인 학원 공지만 |
| GET | /api/v1/academies/my/announcements | STUDENT/TEACHER | 본인 학원 공지만 + read 상태 |
| POST | /api/v1/academies/my/announcements/{id}/read | STUDENT/TEACHER | 본인 학원 공지 read upsert |

`/me` (학원장 관점) vs `/my` (학생·강사 본인 학원 관점) prefix 분리로 의미 구분.

## AcademyMembershipResolver

```java
class AcademyMembershipResolver {
    Long resolveAcademyId(Long accountId, Role role) {
        return switch (role) {
            case OWNER -> ownerAcademy.id
            case TEACHER -> activeAssignment.academyId
            case STUDENT_HIGH, STUDENT_MIDDLE -> activeMembership.academyId
            case MASTER -> null  // master 는 별도 endpoint
        };
    }
}
```

학원 미가입 (학생 PENDING_CONNECT / 강사 unassigned / 학원장 학원 없음) → empty academyId → 공지 목록 빈 응답.

## 회귀 라인

`AnnouncementApiTest`:
1. OWNER 공지 작성 → 201
2. OWNER 본인 공지 목록 → 1 row
3. STUDENT (ACTIVE membership) 본인 학원 공지 → 1 + read=false
4. STUDENT POST /read → read=true (upsert idempotent)
5. 학원 미가입 STUDENT → 빈 목록
6. 다른 학원 STUDENT → 본인 학원 공지만 (학원 격리)
7. TEACHER (ACTIVE assignment) → 본인 학원 공지
8. 다른 OWNER 가 우리 학원 공지 PATCH → 403
9. OWNER 본인 공지 DELETE → 204 + read row CASCADE 삭제
10. STUDENT 작성 시도 → 403

## 관련

- [[decision-academy-management-owner-policy]] — OWNER 권한 매트릭스
- [[decision-teacher-invite-assignment]] — TEACHER assignment 본인 학원 ID 해결
- [[decision-academy-connect-membership-flow]] — STUDENT membership 본인 학원 ID 해결
