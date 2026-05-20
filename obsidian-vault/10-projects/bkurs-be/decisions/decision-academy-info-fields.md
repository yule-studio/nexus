---
title: "B4A 결정 — 학원 정보 필드 (phone/address/logo/website) + 학생·강사 조회 endpoint"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T08:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, academy, info-fields, b4a]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-management-owner-policy]]"
  - "[[decision-announcement-read-tracking]]"
home_hub: bkurs-be
---

# B4A 결정 — 학원 정보 필드 + 학생·강사 조회

## 결론

1. **academy 테이블에 phone / address / logo_url / website 4 컬럼 추가** — nullable, 검증은 application 레벨 (Bean Validation).
2. **PATCH `/api/v1/academies/me`** 가 4 필드 모두 받음 — `null` 이면 변경 없음 (entity.updateInfo invariant).
3. **신규 GET `/api/v1/academies/my/academy`** — 학생/강사 본인 학원 정보 조회. `AcademyMembershipResolver` 재사용.
4. **logo_url 은 URL 문자열만 저장** — 파일 업로드는 B4C+ (별도 PR).

## 왜 필요한가

- bkurs-fe 가 학원 명패 (학원명/로고/연락처) UI 표시 필요 — 현재는 academy.name 만 있음.
- 학원장이 학원 정보 (전화/주소/website) 직접 변경 가능해야 함 — 운영 도구.
- 학생·강사 가 본인 학원 정보 조회 가능해야 함 — 공지(B3C) 와 짝.

## 안 하면 뭐가 문제

- 학원 정보 변경 endpoint 없으면 학원장이 운영자에게 요청 → 운영 부담.
- 학생/강사 본인 학원 정보 조회 못하면 bkurs-fe 가 학원명만 표시 → 학원 정체성 약함.
- 로고 / 주소 등 없으면 학원 페이지 / 공지 / 결제 영수증 등 향후 기능 모두 불완전.

## 4 대안 비교 — 데이터 모델

### A. academy 테이블에 nullable 컬럼 4개 추가 — **채택**

- 장점: 단순. JOIN 불필요. PATCH endpoint 1개로 통합.
- 단점: 컬럼 추가될수록 row 비대화. 향후 academy_metadata key-value 분리 검토 가능.
- **채택 사유**: B4A 4 필드는 small. 정규화 비용 < 단순성 이득.

### B. academy_info 별도 테이블 (1:1)

- 단점: 매 query 마다 JOIN. lazy load 시 N+1 위험.
- **기각**.

### C. academy_metadata key-value (NoSQL 스타일)

- 단점: 검증 분산. 컬럼 단위 query 어려움.
- **기각**.

### D. JSON 컬럼 (academy.info_json)

- 단점: 검증 / 마이그레이션 / 인덱스 모두 어려움.
- **기각**.

## 4 대안 비교 — 학생·강사 조회 권한

### a. 학원장만 GET — 기각

- 단점: bkurs-fe 학생·강사 화면이 학원 정보 못 가져옴 → 학원 정체성 약함.
- **기각**.

### b. AcademyMembershipResolver 재사용 GET `/my/academy` — **채택**

- 장점: B3C 의 resolver 그대로 활용. role 별 본인 학원 통합.
- 단점: 학원 미가입 사용자는 404 (B3C 공지는 빈 목록인데 정책 일관성 검토 필요).
- **채택**: 공지(목록) vs 학원 정보(단일) 다르므로 단일은 404 가 자연스러움.

### c. 모든 인증 사용자에게 학원 정보 공개 (학원 ID 알면 조회 가능)

- 단점: enumeration 가능 (학원 ID 1~N 조회로 모든 학원 정보 수집).
- **기각**.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| PATCH | /api/v1/academies/me | OWNER | 본인 학원 (B3A 기존) — 4 필드 추가 |
| GET | /api/v1/academies/my/academy | ALL (소속 학원 있는 사용자) | Resolver 가 academyId 해결, 없으면 404 |

`/me` (소유 학원 변경) vs `/my/academy` (본인 학원 조회) — B3C 의 prefix 정책 그대로.

## Bean Validation 정책

| 필드 | 검증 |
|---|---|
| phone | `@Size(max=32)` + 형식은 자유 (국가 코드 / 하이픈 / 괄호 등 다양) |
| address | `@Size(max=500)` |
| logoUrl | `@Size(max=500)` + URL pattern (간단한 `^https?://` 만, 엄격 검증은 클라이언트) |
| website | `@Size(max=500)` + URL pattern |

엄격한 URL/phone 검증은 false positive 위험 — application 레벨 최소 검증, 클라이언트가 UX 검증.

## 회귀 라인

`AcademyInfoApiTest`:
1. PATCH name 만 변경 → 200 + phone null 유지 (기존 회귀 동등)
2. PATCH phone 만 변경 → 200 + name 유지
3. PATCH 4 필드 동시 → 200 + 모두 반영
4. 학생 GET /my/academy → 본인 학원 정보 (logoUrl/phone 포함)
5. 강사 GET /my/academy → 본인 학원 정보
6. 학원 미가입 학생 GET → 404
7. PATCH 잘못된 URL (예: "not-a-url") → 400 (Bean Validation)
8. PATCH 빈 객체 → 200 + 변경 없음

## 관련

- [[decision-academy-management-owner-policy]] — Academy entity 의 owner policy
- [[decision-announcement-read-tracking]] — AcademyMembershipResolver 재사용
