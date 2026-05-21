---
title: "B3A 결정 — 학원 CRUD + 코드 관리 정책 (1인 1학원 + 다중 active code)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T06:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, academy, owner, code-management, b3a]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-connect-membership-flow]]"
home_hub: bkurs-be
---

# B3A 결정 — 학원 CRUD + 코드 관리 정책 (1인 1학원 + 다중 active code)

## 결론

1. **학원장 1인 = 학원 1개** (B3A 기준). 다중 학원 운영 / 학원 이관은 B4+.
2. **학원 코드 다중 active 허용** — 학원장이 여러 active code 동시 운영 가능 (예: 분기별 신학기 코드 / 강사용 별도 코드).
3. **코드 자동 생성** — 학원장은 요청만, 서버가 충돌 없는 random code 생성 (영문대문자+숫자 8자, 충돌 시 재시도 최대 5회).
4. **권한 매트릭스**:
   - POST `/academies` — OWNER 만, 본인 학원 없는 OWNER 만 (409 if exists)
   - PATCH `/academies/me` — 본인 학원만
   - POST `/academies/me/codes` / DELETE — 본인 학원만
   - GET `/academies/me/memberships/pending` — OWNER 본인 학원 큐

## 왜 필요한가

- B2C 까지는 academy / academy_code 를 테스트 fixture 로만 만들었다. 운영 진입을 위해 학원장이 직접 학원 생성 + 코드 발급 + 비활성화 가능해야 함.
- "1인 1학원" 정책은 B3A 의 최소 가정. 학원 이관 (학원장 교체) 이나 다중 학원 운영자는 운영 정책이 명확해진 후 (B4+).
- 다중 active code 허용: 학원 운영 현실 — 같은 학원에 여러 시기/대상의 코드 발급이 흔하다. 단일 active 제약은 학원장 UX 깨짐.

## 안 하면 뭐가 문제

- 학원 생성 endpoint 없으면 운영 도구가 SQL 직접 입력 또는 fixture migration → 운영 시 위험.
- 코드 발급 API 없으면 학원장이 학생 가입 시 코드 알려줄 수 없음 → B2C connect flow 작동 불가.
- 권한 분리 없으면 다른 학원장이 우리 학원 코드 발급 가능 = 보안 사고.

## 4 대안 비교 — 학원 1:N 관계

### A. 1인 1학원 (현 B2C 가정) — **B3A 채택**

- 장점: 단순. UI 도 단순 (학원장 = 학원 1:1 매핑).
- 단점: 향후 분원 / 다중 학원 운영자 시나리오 시 schema 변경 필요.
- **채택 사유**: bkurs MVP 의 학원장 1인 가정. B4+ 에서 academy_member 테이블 도입으로 확장.

### B. 학원장 N개 학원 허용

- 장점: 분원 / 다중 학원 즉시 지원.
- 단점: 권한 / UX / "내 학원" 선택 흐름 추가 — B3A 가 비대해짐.
- **기각 사유**: MVP 단계에서 over-design.

### C. 학원에 OWNER 다중 (academy_member 테이블)

- 장점: 학원 측 입장에서 운영자 위임 가능 (학원장 + 부원장 등).
- 단점: 권한 모델 / endpoint 복잡도 폭증.
- **기각 사유**: B3A 범위 초과. 추후 academy_member 도입 시 검토.

### D. 학원 ↔ OWNER N:M 완전 자유

- 단점: A + B + C 의 단점 모두.
- **기각 사유**: bkurs 운영 현실과 불일치.

## 4 대안 비교 — 코드 정책

### a. 학원당 단일 active code (B2C 가정) — 기각

- 장점: 단순.
- 단점: 학원 운영 현실과 불일치 (분기별 / 강사용 / 신학기 / 보충 등 여러 코드).
- **기각 사유**: 학원장 UX 제약.

### b. 다중 active 허용, 학원장이 발급/비활성화 직접 — **채택**

- 장점: 운영 현실 그대로. 학원장이 코드 별 라벨 / 만료 / max_uses 도 향후 추가 가능.
- 단점: 코드 충돌 (전역 uk_academy_code) 처리 — random 생성 + 재시도.
- **채택 사유**: 운영 도구로 사용 가능한 minimal 형태.

### c. 코드 자동 만료 (TTL 기반)

- 단점: B3A 범위 초과. 만료 정책 / 알림 / 갱신 UX 모두 별도 결정.
- **기각 사유**: 후속 PR (B3 후반 또는 B4) 로 분리.

### d. 1회용 (max_uses=1) 코드

- 단점: 학원 운영 현실 불일치 (한 코드를 학년 단위로 공유).
- **기각 사유**: 향후 옵션으로 추가 가능.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies | OWNER | 본인 학원 없음 |
| GET | /api/v1/academies/me | OWNER | 본인 학원 있음 |
| PATCH | /api/v1/academies/me | OWNER | 본인 학원만 |
| POST | /api/v1/academies/me/codes | OWNER | 본인 학원만 |
| GET | /api/v1/academies/me/codes | OWNER | 본인 학원만 |
| POST | /api/v1/academies/me/codes/{codeId}/deactivate | OWNER | 본인 학원 코드만 |
| GET | /api/v1/academies/me/memberships/pending | OWNER | 본인 학원 큐 |

`/academies/memberships/{id}/approve|reject` (B2C) 는 그대로 유지.

## 코드 자동 생성 알고리즘

```java
// 영문대문자(A-Z) + 숫자(0-9) 8자, ambiguous 문자 제외 (0, O, I, 1)
private static final char[] ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789".toCharArray();

private String generate() {
    char[] chars = new char[8];
    for (int i = 0; i < 8; i++) {
        chars[i] = ALPHABET[random.nextInt(ALPHABET.length)];
    }
    return new String(chars);
}
```

- alphabet 32 chars → 32^8 ≈ 1.1 trillion 조합 → 학원 1000 개 × 코드 100개 = 10만 코드에서 충돌 확률 무시 가능 (생일문제 계산 시 < 1e-7).
- DB UNIQUE 제약으로 충돌 시 ConstraintViolationException → 서비스에서 catch 후 최대 5회 재시도.

## 회귀 라인

`AcademyManagementApiTest`:

1. OWNER 가 학원 생성 (본인 학원 없음) → 201
2. OWNER 가 두 번째 학원 생성 → 409 (`academy_already_exists`)
3. STUDENT 가 학원 생성 → 403 (@PreAuthorize)
4. GET /academies/me → 본인 학원
5. PATCH name 변경 → 200
6. POST 코드 발급 → 201 + 새 row + active=true
7. 같은 학원에 코드 2개 발급 → 둘 다 active
8. POST deactivate → active=false
9. GET 코드 목록 → 2 rows
10. 다른 학원장이 다른 학원의 코드 deactivate 시도 → 403 (`not_your_academy`)

## 관련

- [[decision-academy-connect-membership-flow]] — B2C 의 connect flow 가 본 PR 의 코드 발급 endpoint 와 결합
- [[../bkurs-be]] — PR 시리즈 위치 (B3A)
