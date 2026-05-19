---
title: "Account 테이블 전략 — Single Table + Role enum + Status enum + Profile 분리"
kind: decision
project: bkurs-be
session_id: bkurs-be-b2-auth
created_at: 2026-05-20T01:00:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - "[[../bkurs-be]]"
  - "[[decision-auth-architecture-jwt-refresh]]"
  - "[[../task-logs/task-log-b2a-account-jwt]]"
  - "[[../../../20-areas/backend/spring/java-spring/database/jpa]]"
tags: [decision, entity, account, role, status, b2]
---

# 목표

4 role (학생 고등/중등/강사/원장/마스터) 의 인증 / 인가 단일 모델 + 각 role 별 도메인 정보를 어떻게 분리할 것인가.

# 결정

**Single Table (`account`) + Role enum + Status enum** + role 별 **Profile 테이블 분리** (1:1).

```
account                       student_profile                 teacher_profile
─────────                     ────────────────                ────────────────
id PK                  ←──    account_id PK,FK          ←──   account_id PK,FK
email UQ                      school_type (HIGH/MIDDLE)        academy_id FK
password_hash                 grade                            role_in_academy
role (enum)                   school_name                      ...
status (enum)                 parent_phone
created_at                    academy_id FK (nullable)
updated_at                    ...
```

`account` = 인증 + 기본 식별만. role 별 추가 정보는 `student_profile` / `teacher_profile` / `owner_profile` 에 1:1.

## Role enum

```java
public enum Role {
    STUDENT_HIGH,    // 고등 학생
    STUDENT_MIDDLE,  // 중등 학생
    TEACHER,         // 강사
    OWNER,           // 학원장
    MASTER           // 본사 (시스템 관리자)
}
```

## AccountStatus enum

```java
public enum AccountStatus {
    PENDING_CONNECT,   // 가입 직후 — 학생만. 학원 코드 미입력
    PENDING_APPROVE,   // 학원 코드 입력 후 — 학원 측 승인 대기
    ACTIVE,            // 정상 사용 가능
    REJECTED,          // 학원 측 거절 (사유 별도 컬럼)
    BLOCKED            // 마스터 관리자가 정지
}
```

# 대안

| 옵션 | 장점 | 단점 | 결정 |
|---|---|---|---|
| **Single Table + Role enum + Profile 분리 (선택)** | 인증 한 곳 / role 별 정보 깔끔 분리 / role 추가 시 신규 profile 만 추가 | profile 테이블 N+1 가능성 (해결: fetch join) | ✅ |
| Single Table Inheritance (JPA `@Inheritance(SINGLE_TABLE)`) — 한 테이블에 모든 컬럼 + discriminator | JPA 표준 패턴, 쿼리 한 번 | 컬럼 nullable 폭증 (학생만 쓰는 grade 가 강사 row 에선 NULL), index 효율 ↓ | ⛔ |
| Joined Inheritance (`@Inheritance(JOINED)`) — 부모 + 자식 테이블, 자식이 부모 PK 공유 | 정규화 깔끔 | 조인 강제 (모든 SELECT 마다), 부모 PK 충돌 시 마이그레이션 복잡 | ⛔ (Profile 분리와 효과 비슷 + JPA 제약 큼) |
| 완전 별도 테이블 (student / teacher / owner — 인증 정보 각자) | role 간 완전 격리 | 이메일 unique 강제 어려움 (중복 가입 가능), 로그인 endpoint 가 role 별 분기 | ⛔ |

# Why (4 구조)

## 왜 필요한가

- 인증은 4 role 모두 동일 (이메일 + 비번 + role 식별) → `account` 한 곳
- 도메인 정보는 role 별 다름 (학생 학년 / 강사 학원ID / 원장 사업자번호) → profile 분리
- role 추가 (예: 학부모 PARENT) 시 profile 만 추가, `account` 는 enum 한 줄 + migration 한 줄

## 안 하면 무엇이 깨지는가

Single Table Inheritance 로 다 합치면:
- `account` 테이블에 grade / school_name / parent_phone / academy_id / 사업자번호 ... 다 nullable
- 학생 row 의 90% 컬럼이 NULL — 쿼리 plan 비효율
- 새 role 추가 시 컬럼 추가 → `ddl-auto: validate` 통과 위해 Flyway V<N>__ 마이그레이션 매번
- profile 변경이 인증 테이블에 영향 — 변경 위험성 ↑

완전 별도 테이블로 가면:
- `student.email`, `teacher.email`, `owner.email` 각자 unique → cross-table unique 강제 어려움 (같은 이메일로 학생/강사 동시 가입 가능해짐)
- 로그인 endpoint 가 role 별 분기 → AuthController 복잡

## 대안 비교

위 표.

## 트레이드오프

- profile 별도 테이블 → 학생 정보 조회 시 join 1회 추가 (해결: `@EntityGraph` 또는 fetch join)
- migration 이 account + profile 동시 변경되므로 V<N> 1개에 두 테이블 묶기

# 산출물 (B2A — 본 PR)

- `account` 테이블만 (B2A 범위)
- Role enum 5종, AccountStatus enum 5종
- `Account` entity + `AccountRepository`
- profile 테이블은 도메인 PR 에서 (B3 Academy / B4 Student / ...)

# 인덱스 / 제약

```sql
CREATE TABLE account (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(32) NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'PENDING_CONNECT',
    created_at DATETIME(6) NOT NULL,
    updated_at DATETIME(6) NOT NULL,
    CONSTRAINT uk_account_email UNIQUE (email),
    INDEX idx_account_status (status),
    INDEX idx_account_role_status (role, status)  -- 학원이 본인 role 의 PENDING_APPROVE 만 조회 빈번
) ENGINE=InnoDB CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

# 참고

- [[../bkurs-be|프로젝트 hub]]
- [[decision-auth-architecture-jwt-refresh|JWT + Refresh 결정]]
- [[../../../20-areas/backend/spring/java-spring/database/jpa|JPA 노트]]
