---
title: bkurs-fe — Hook 디렉터리 컨벤션
kind: decision
status: accepted
created_at: 2026-05-21
tags: [bkurs-fe, convention, hooks, react-query]
related:
  - "[[architecture-api-foundation]]"
  - "[[bkurs-fe]]"
  - "[[decision-vite-react-19-tailwind-antd-jotai]]"
home_hub: "[[bkurs-fe]]"
---

# bkurs-fe — Hook 디렉터리 컨벤션

## 결정

**page-local `use*.ts` 패턴** 을 표준으로 유지. cross-domain 공유 hook 만
`src/lib/hooks/` 에 둔다. inroad-high 의 `_pages/<role>/_hooks/` 패턴은
**채택하지 않는다**.

| 위치 | 용도 | 예시 |
|------|------|------|
| `src/lib/hooks/` | 2 role 이상이 공유 — apiClient 위 react-query wrapping | `useConceptDiagnosis` (학생/강사 공용) |
| `src/pages/<role>/_pages/<feature>/use*.ts` | 화면 1개 (또는 같은 feature 내) 전용 | `useAdminLogin`, `useMasterLogin`, `useUploadFlow`, `useLessons` |
| `src/pages/<role>/_hooks/` | **금지** — 채택하지 않음 | (inroad-high 원본 패턴) |

## 왜 이렇게 결정했나

### 1. 기존 컨벤션과 충돌
bkurs-fe S2N (PR #28까지) 동안 모든 hook 이 page-local `use*.ts`:
- `admin/_pages/login/useAdminLogin.ts`
- `master/_pages/login/useMasterLogin.ts`
- `master/_pages/lessons/useLessons.ts`
- `master/_pages/lessons/useUploadFlow.ts`

여기에 `_pages/<role>/_hooks/` 디렉터리를 도입하면 **두 컨벤션이 공존** —
신규 작업자가 "어디에 둬야 하나" 매번 결정해야 함. 한 가지로 통일이
유지 비용 낮음.

### 2. cross-domain 공유는 `src/lib/hooks/` 가 자연스러움
`useConceptDiagnosis` 는 학생 화면 + 강사 화면 양쪽에서 사용 — page-local
이면 어느 role 의 page-local 인지 모호.

`src/lib/hooks/` 는 이미 S2 PR #65 에서 시작됨. 패턴 일관성 유지.

### 3. inroad-high 원본은 supabase 직접 호출 hook 묶음
inroad-high 의 `_hooks/useHighSuhaengSubmission.ts` 등은 supabase + react-query
조합. bkurs-fe 는 BE 호출로 전환 중 — supabase hook 을 그대로 가져오면
S3 cycle 에서 다시 들어내야 함. **bkurs-fe 의 hook 은 BE 호출 기반으로
새로 작성** — 패키지 위치도 그에 맞게 `src/lib/hooks/`.

### 4. 대안 검토
- **`_pages/<role>/_hooks/` 채택**: inroad-high 원본 패턴 따라가서 mapping
  쉬움. 단점 (1) 두 컨벤션 공존 (2) supabase hook 의 짐이 디렉터리 자체에
  스며듦. **기각**.
- **모두 `src/lib/hooks/`**: 1 화면 전용 hook 까지 lib 으로 옮기면 page 단위
  응집도 깨짐. login/upload 같은 1 화면 hook 은 page-local 이 더 자연스러움.
  **기각**.
- **(채택) page-local + lib/hooks 의 2 단계 구조**: 응집도 + 공유 양쪽 만족.

## 적용 맥락

### 새 hook 추가 절차

1. **1 화면 (또는 같은 feature 내) 전용?**
   - YES → `src/pages/<role>/_pages/<feature>/use<Name>.ts`.
   - NO → 다음 질문.
2. **2 role 이상 (학생/강사/관리자) 공유?**
   - YES → `src/lib/hooks/use<Name>.ts` + `src/lib/hooks/index.ts` 에 export.
3. **react-query 사용 시**:
   - 쿼리 키: `['<domain>', '<scope>']` (예: `['concept', 'mine']`).
   - mutation onSuccess: `setQueryData` 우선 (라운드 트립 절약).
   - 외부 데이터 fetch 필수면 `enabled` 로 placeholder 회피.

### Cycle 3 진행 시 (suhaeng/saenggibu)

BE 의 suhaeng/saenggibu 도메인이 추가되면:
- `src/lib/api/suhaeng.ts` — domain client (B30 concept 패턴).
- `src/lib/hooks/useSuhaeng.ts` — react-query hook (cross-role 공유).
- 화면이 hook 호출 — `_hooks/` 디렉터리 도입 X.

## 관련 노트

- [[architecture-api-foundation]] — apiClient + envelope + S1/S2/S3 단계
- [[bkurs-fe]] — 프로젝트 허브
- [[decision-vite-react-19-tailwind-antd-jotai]] — 초기 기술 스택 결정

## 참고

- 코드 예시: `bkurs-fe/src/lib/hooks/useConceptDiagnosis.ts` (S2 #64 표준)
- page-local 예시: `bkurs-fe/src/pages/master/_pages/lessons/useLessons.ts`
- 후속: Cycle 3 진행 시 본 ADR 준수 (suhaeng/saenggibu hook 은 `src/lib/hooks/`).
