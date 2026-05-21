---
title: "bkurs-fe — 공유 컴포넌트 매핑"
kind: handover
status: stable
created_at: 2026-05-21T23:00:00+09:00
tags: [bkurs-fe, handover, components, reuse]
related:
  - "[[bkurs-fe]]"
  - "[[role-modules]]"
  - "[[directory-structure]]"
home_hub: bkurs-fe
---

# bkurs-fe — 공유 컴포넌트 매핑

> "이 패널 어디 또 쓰지" / "왜 같은 컴포넌트가 두 디렉토리에 있지" — 핵심 재사용 패턴 정리.
> inroad-high 의 senior 패턴 (`AiAnalysisPanel 6회 재사용` 등) 을 그대로 보존하면서 디자인
> variation 은 role 별로 분리.

## 0. 결론 표

| 컴포넌트 | 위치 | 재사용 횟수 | 동일 디자인? |
| --- | --- | --- | --- |
| `AiAnalysisPanel` | high-tabs/expect 안 (admin 학생 상세) | 6 | 화면 단위로 instance |
| `ThreadView` | high-student/_components/ + middle-student/_components/ | 7 (high 3 + middle 4) | role 별 별도 (디자인 약간 다름) |
| `ScoreRadar` | middle-tabs/middle-past 안 | 2 | middle 한정 — 동일 |
| `RootLayout` | components/layout/ | 전역 1 | role-무관 outer shell |

## 1. AiAnalysisPanel

AI 분석 결과를 카드 형태로 보여주는 패널. 5 단계 별 요약 + 점수 + 코멘트.

### 실제 위치 (코드 SSoT)

`src/pages/admin/_pages/students/detail/high-tabs/expect/AiAnalysisPanel.tsx`

> ⚠️ 이름이 admin 디렉토리에 있지만 학생 본인 페이지에서도 import — 공유 SSoT. 향후 `_shared/` 로 승격이 더 자연스러움 (현재는 inroad-high 의 위치 그대로).

### 사용처 (6)

| 페이지 | role | 비고 |
| --- | --- | --- |
| `admin/students/detail/high-tabs/expect/` | OWNER | 학원장이 학생 expect 결과 검토 |
| `admin/students/detail/high-tabs/past/` | OWNER | 학원장이 past 결과 검토 |
| `admin/students/detail/middle-tabs/middle-expect/` | OWNER | |
| `admin/students/detail/middle-tabs/middle-past/` | OWNER | + ScoreRadar 동반 |
| `admin/students/detail/middle-tabs/middle-presentation/` | OWNER | + ScoreRadar 동반 |
| `admin/students/detail/middle-tabs/middle-suhaeng/` | OWNER | |

학생 본인 페이지 (`high-student/_pages/expect/Expect.tsx` 등) 도 본 컴포넌트를 import — 같은
디자인.

### 타입

`AIAnalysisResult` (5 단계 점수 + 코멘트) — `_data/student-mock.ts` 또는 BE 응답에서 받음.

> 중복 정의 0 — 타입은 mock 파일 또는 lib/api 의 response 타입에서 import. PR review 시 새로
> 정의하는 경우 reject.

## 2. ThreadView

학생 ↔ 강사 5 단계 답변 thread. expand / collapse + 단계별 status (DRAFT / SUBMITTED / FEEDBACK / REVISED / FINAL).

### 위치 — role 별 분리

- `src/pages/high-student/_components/ThreadView.tsx`
- `src/pages/middle-student/_components/ThreadView.tsx`

**별도 instance인 이유**: brand 색 (high=파랑, middle=초록), 사이드바 너비, 텍스트 톤 (반말/존대)
이 role 별로 다름. inroad-high 의 분기 그대로 보존.

> 디자인이 통일되면 `_shared/ThreadView.tsx` 로 승격 후보. props 로 `brand: 'high' | 'middle'`
> 인자 받아 swap.

### 사용처 (7)

| 페이지 | role | 단계 |
| --- | --- | --- |
| `high-student/_pages/expect/` | STUDENT_HIGH | 예상 문항 답변 thread |
| `high-student/_pages/past/` | STUDENT_HIGH | 기출 답변 thread |
| `high-student/_pages/presentation/` | STUDENT_HIGH | 발표 thread |
| `middle-student/_pages/expect/` | STUDENT_MIDDLE | |
| `middle-student/_pages/past/` | STUDENT_MIDDLE | |
| `middle-student/_pages/presentation/` | STUDENT_MIDDLE | |
| `middle-student/_pages/suhaeng/` | STUDENT_MIDDLE | suhaeng round thread |

### 타입

`StudentAnswerThread` — 5 단계 답변 record. high / middle 동일 타입 — 중복 정의 0.

## 3. ScoreRadar

5 영역 (논리 / 표현 / 사고력 / 창의 / 일관성 등) radar chart. `recharts` 의 `RadarChart` wrap.

### 위치

`src/pages/admin/_pages/students/detail/middle-tabs/middle-past/ScoreRadar.tsx`

### 사용처 (2)

| 페이지 | role |
| --- | --- |
| `admin/students/detail/middle-tabs/middle-past/` | OWNER (학원장 학생 상세) |
| `admin/students/detail/middle-tabs/middle-presentation/` | OWNER |

middle 한정 — high 측 동일 시각화는 후속 (또는 별도 디자인).

### 타입

`ScoreItem` (영역 + score) — middle-past/presentation 양쪽 공유.

## 4. RootLayout

`src/components/layout/RootLayout.tsx` — 모든 라우트의 outer shell (`/` index 의 wrapper). 거의
빈 layer — 향후 글로벌 toast / banner / error boundary 자리.

## 5. role 한정 컴포넌트 (`{role}/_components/`)

각 role 의 `_components/` 는 **본 role 내** 2개 이상 페이지에서 재사용되는 것만.

현재:
- `high-student/_components/ThreadView.tsx` — high 3 페이지 재사용
- `middle-student/_components/ThreadView.tsx` — middle 4 페이지 재사용

admin / master 는 현재 `_components/` 없음 — 페이지 내부에서 sub-component 화 (inline). 재사용 발생 시 `_components/` 신설.

## 6. inline sub-component (페이지 내부)

페이지 폴더 안에 작은 컴포넌트들 두는 패턴.

예: `high-student/_pages/expect/`:
- `Expect.tsx` (라우트 main)
- `ExpectHeader.tsx`, `ExpectList.tsx`, `ExpectThread.tsx` (sub)

규칙:
- 본 페이지에서만 사용 — 다른 페이지 import 금지.
- 다른 페이지에서도 필요해지면 `_components/` 로 승격 + import path 갱신.
- 페이지 main 이 200줄 넘으면 sub 분리 검토.

## 7. 신규 공유 컴포넌트 추가 절차

1. **위치 결정**:
   - 1 페이지만 사용 → inline (페이지 폴더 내)
   - 1 role 내 2+ 페이지 → `{role}/_components/`
   - 2+ role → `_shared/` 또는 `components/` (top)
2. props 로 variation 처리 — role 별 분기를 component 내부에 두지 않음 (예: `brand: 'high' | 'middle'`).
3. 타입은 lib/api 의 response 타입 또는 별도 typings — 중복 정의 금지.
4. Storybook 등은 현재 없음 — JSDoc 한 줄 + props 명확하게.

## 8. 디자인 토큰만 사용

모든 컴포넌트는 `tailwind.config.js` 의 brand-* 토큰만 사용. raw hex (`#2563EB` 등) 직접 금지.
inroad-high 의 디자인 보존이 본 프로젝트의 핵심 제약 ([[design-system]] 참조).

## cross-link

- [[bkurs-fe]]
- [[directory-structure]] — components / pages 위치
- [[role-modules]] — role 별 페이지 책임 (사용처와 연결)
- [[design-system]] — brand 토큰
- [[lib-layer-guide]] — 타입 SSoT
