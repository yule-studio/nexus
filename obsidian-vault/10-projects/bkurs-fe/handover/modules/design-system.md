---
title: "bkurs-fe — 디자인 시스템"
kind: handover
status: stable
created_at: 2026-05-21T23:10:00+09:00
tags: [bkurs-fe, handover, design, tailwind, antd]
related:
  - "[[bkurs-fe]]"
  - "[[directory-structure]]"
  - "[[shared-components]]"
  - "[[../../memory/project_bkurs_fe_reimplementation]]"
home_hub: bkurs-fe
---

# bkurs-fe — 디자인 시스템

> Tailwind brand 토큰 + AntD ThemeConfig 가 **두 곳에서 일치**해야 한다. inroad-high
> (camshf-crypto) 의 디자인을 **1:1 보존**하는 것이 본 프로젝트의 핵심 제약 —
> [[../../memory/project_bkurs_fe_reimplementation]] 참조.

## 1. 한 장 요약

```
tailwind.config.js              styles/antd-theme.ts
  ├─ brand-high   #2563EB         token.colorPrimary  #2563EB  ← 일치
  ├─ brand-middle #10B981         token.colorSuccess  #10B981  ← 일치
  ├─ brand-master #7C3AED         (master 페이지 한정 — Tailwind class)
  └─ ink/line/...                 colorTextBase/colorBorder    ← 일치
```

## 2. role 별 brand 색

| role | 메인 컬러 | 사용처 |
| --- | --- | --- |
| high-student | brand-high `#2563EB` (파랑) | 고등 학생 사이드바 / 버튼 / 강조 |
| middle-student | brand-middle `#10B981` (초록) | 중등 학생 |
| master | brand-master `#7C3AED` (보라) | 본사 사이드바 / 헤더 |
| admin | brand-primary `#2563EB` (파랑) — high 와 같은 톤 | 학원장 |

신규 페이지의 강조 색은 **role brand 만** 사용. raw hex 직접 금지.

## 3. Tailwind 토큰 (`tailwind.config.js`)

### 3.1 colors

```js
colors: {
  // 파랑 (고등 / B2B / 어드민)
  brand: {
    primary: '#2563EB',
    'primary-hover': '#1D4ED8',
    'primary-dark': '#1E3A8A',
    'primary-light': '#DBEAFE',
    'primary-bg': '#EFF6FF',
    'gradient-start': '#1E40AF',
    'gradient-end': '#3B82F6',
  },
  'brand-high': {
    DEFAULT: '#2563EB',
    hover: '#1D4ED8',
    dark: '#1E3A8A',
    light: '#93C5FD',
    bg: '#DBEAFE',
    pale: '#EFF6FF',
  },
  'brand-middle': {
    DEFAULT: '#10B981',
    hover: '#059669',
    dark: '#047857',
    light: '#A7F3D0',
    bg: '#D1FAE5',
    pale: '#ECFDF5',
  },
  'brand-master': {
    DEFAULT: '#7C3AED',
    hover: '#6D28D9',
    dark: '#5B21B6',
    light: '#C4B5FD',
    bg: '#F5F3FF',
    pale: '#FAF5FF',
    gradientStart: '#5B21B6',
    gradientEnd: '#8B5CF6',
  },
  ink: { DEFAULT: '#0F172A', secondary: '#475569', muted: '#94A3B8' },
  line: { DEFAULT: '#E2E8F0', light: '#F1F5F9' },
}
```

### 3.2 사용 예

```tsx
<button className="bg-brand-high text-white hover:bg-brand-high-hover">제출</button>
<div className="bg-brand-master-pale text-ink">본사 알림</div>
<aside className="border-r border-line"></aside>
```

scheme:
- `DEFAULT` (단독 사용) — `bg-brand-high`
- `hover`, `dark`, `light`, `bg`, `pale` — variation
- `brand-{role}-{variant}` 형태 일관

### 3.3 fonts

```js
fontFamily: {
  sans: ['"Pretendard Variable"', 'Pretendard', 'system-ui', 'sans-serif'],
},
```

Pretendard 사용 — index.html / public 폰트 로딩 또는 CDN.

### 3.4 corePlugins.preflight

```js
corePlugins: { preflight: true }
```

Tailwind preflight 활성 — 단, AntD 컴포넌트 (`<Button>`) 는 AntD 가 우선 (cascade 순서로).

## 4. AntD ThemeConfig (`src/styles/antd-theme.ts`)

```ts
export const antdTheme: ThemeConfig = {
  token: {
    colorPrimary:  '#2563EB',  // = brand-high.DEFAULT
    colorInfo:     '#2563EB',
    colorSuccess:  '#10B981',  // = brand-middle.DEFAULT
    colorWarning:  '#F59E0B',
    colorError:    '#EF4444',
    colorTextBase: '#0F172A',  // = ink.DEFAULT
    colorBgBase:   '#FFFFFF',
    colorBorder:   '#E2E8F0',  // = line.DEFAULT
    borderRadius:  8,
    fontFamily:    '"Pretendard Variable", ...',
  },
  components: {
    Button: { controlHeight: 40, borderRadius: 10, fontWeight: 600 },
    Input:  { controlHeight: 40, borderRadius: 8 },
    Select: { controlHeight: 40, borderRadius: 8 },
  },
}
```

`AppProviders.tsx`:

```tsx
<ConfigProvider theme={antdTheme} locale={koKR}>
  {children}
</ConfigProvider>
```

**locale=ko_KR** — DatePicker, Pagination, Form validation message 한국어.

## 5. 일관성 원칙

| 항목 | Tailwind | AntD ThemeConfig |
| --- | --- | --- |
| primary (파랑) | brand-high.DEFAULT `#2563EB` | colorPrimary `#2563EB` |
| success (초록) | brand-middle.DEFAULT `#10B981` | colorSuccess `#10B981` |
| 텍스트 베이스 | ink.DEFAULT `#0F172A` | colorTextBase `#0F172A` |
| 보더 | line.DEFAULT `#E2E8F0` | colorBorder `#E2E8F0` |
| 폰트 | Pretendard Variable | Pretendard Variable |
| 라운드 (기본) | 8px (`rounded`) | borderRadius 8 |
| 버튼 height | (custom) | controlHeight 40 |

**두 곳을 동시에 갱신**하지 않으면 컴포넌트 종류 (Tailwind div vs AntD Button) 별로 색이 어긋남.

## 6. 신규 색 / 토큰 추가 금지

원본 inroad-high 의 디자인을 보존. 새 brand color 추가 금지. 새 spacing / radius 도 가능한
기존 토큰 재사용.

예외 (PR review 통과 시):
- 명확한 상태 추가 — 예: warning-strong 같은 누락된 variant
- accessibility 보강 — 대비 부족 색 보정

이 경우에도 [[../../memory/project_bkurs_fe_reimplementation]] 의 "디자인 토큰 1:1 유지" 정책에
대한 명시적 예외 사유를 PR body 에.

## 7. raw hex 사용 금지

```tsx
// ❌ 금지
<div style={{ background: '#2563EB' }}>...</div>
<div className="text-[#0F172A]">...</div>

// ✅ 허용
<div className="bg-brand-high text-ink">...</div>
```

예외: 일회성 외부 SDK 색 매핑 (예: 결제 PG 브랜드 색). 이 경우에도 상수로 빼서 한 곳만.

## 8. 디자인 검수 절차

1. 새 페이지 작성 후 inroad-high (`~/masterway-dev/inroad-high`) 와 시각 비교.
2. brand 토큰만 사용했는지 검토 (raw hex grep).
3. AntD 컴포넌트 (`<Button>` `<Input>` `<Modal>`) 와 Tailwind div 가 같은 brand 색으로 보이는지 확인.
4. role 별로 색 swap 확인 (high=파랑, middle=초록 등).

## 9. inroad-high 와의 차이점

| 항목 | inroad-high | bkurs-fe |
| --- | --- | --- |
| 디자인 토큰 | 동일 | 1:1 보존 |
| 폴더 구조 | 기존 (page/admin/_pages...) | 정돈 (alias / strict TS / lib 통합) |
| 백엔드 | Supabase 직접 | bkurs-be REST API + envelope |
| 상태 관리 | Jotai + React Query | 동일 |
| 라우팅 | React Router 7 | 동일 |

## 10. 컴포넌트 디자인 시 체크리스트

- [ ] role 별 brand 토큰 사용
- [ ] raw hex 0
- [ ] AntD `<Button type="primary">` 시 자동 brand 색 (테마로 통일)
- [ ] inroad-high 와 시각 차이 없는지
- [ ] `controlHeight: 40` 의 input 과 일관된 form layout
- [ ] dark mode — **현재 미구현**. 후속 ([[bkurs-fe#5]]).

## cross-link

- [[bkurs-fe]]
- [[directory-structure]] — `styles/antd-theme.ts` + `tailwind.config.js` 위치
- [[shared-components]] — ThreadView role 별 brand 차이
- [[role-modules]] — role ↔ brand 매핑
- [[../../memory/project_bkurs_fe_reimplementation]] — 디자인 보존 정책
