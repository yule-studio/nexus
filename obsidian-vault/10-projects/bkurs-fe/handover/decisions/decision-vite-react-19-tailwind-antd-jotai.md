---
title: "bkurs-fe 기술 스택 결정 — Vite 8 + React 19 + Tailwind + AntD 6 + Jotai + TanStack Query"
kind: decision
project: bkurs-fe
session_id: bkurs-fe-setup
created_at: 2026-05-20T00:25:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - "[[../bkurs-fe]]"
  - "[[../task-logs/task-log-s2-series-summary]]"
  - "[[../../bkurs-be/bkurs-be]]"
tags: [decision, frontend, react, vite, tailwind]
---

# 목표

inroad-high (camshf-crypto 의 운영 React 코드) 를 디자인 동일하게 유지하면서 senior 분해 + 최신 stack 으로 재구성.

# 결정

| 영역 | 결정 | 근거 |
|---|---|---|
| 빌드 | **Vite 8** | inroad-high 가 CRA 였음. Vite 가 빠르고 esbuild / rollup 양쪽 활용 |
| UI 라이브러리 | **React 19** | 최신 LTS, useFormStatus / use 등 새 API |
| 언어 | **TypeScript 6** | 엄격 type, 백엔드 협의 시 타입 계약 명확 |
| CSS | **Tailwind 3.4** | inroad-high 의 inline style 을 utility 로 정리, 디자인 토큰화 (`brand-master` / `brand-high` / `brand-middle`) |
| 컴포넌트 | **Ant Design 6** | 모달 / 폼 / 테이블 빠른 조립 |
| 상태 | **Jotai** | atom 단위 — 인증 / sidebar collapsed 등 작은 단위 적합 |
| 서버 상태 | **TanStack React Query** | mock → API 교체 시 hook 한 줄 |
| 라우팅 | **React Router v7** | createBrowserRouter + 중첩 outlet |
| 차트 | **recharts** | 결제 추이 / KPI 차트 |
| 백엔드 | (분리) [[../../bkurs-be/bkurs-be]] | inroad-high 의 Supabase 직접 호출 패턴 폐기, 별도 API 서버 사용 |

# 대안

- **Next.js** — SSR 필요 없음, SPA 가 충분. 학원 측 사용자만 — SEO 무관. → **기각**
- **CSS-in-JS (styled-components / emotion)** — 디자인 토큰 + utility-first 가 더 자연스러움. → **기각**
- **Redux Toolkit** — 학원 도메인은 단순. Jotai atom 으로 충분. → **기각**
- **Zustand** — 가벼우나 derived atom 패턴이 Jotai 가 더 명확. → **기각**
- **SWR** — TanStack Query 가 mutation / invalidation 등 풍부. → **기각**

# Why (4 구조)

**왜 필요한가**: 처음에 stack 안 고정하면 PR 별로 라이브러리 추가 / 변경 → 번들 사이즈 폭증 + 일관성 깨짐.

**안 하면 무엇이 깨지는가**: 28 PR 중 후반 (S2M~) 에 와서 차트 라이브러리 바꾸면 — recharts 사용 8 곳 모두 수정. mock 데이터 모양도 함께 변경.

**대안 비교**: 위 표.

**트레이드오프**: AntD 와 Tailwind 동시 사용 — 클래스 충돌 가능성. AntD 컴포넌트 안쪽은 AntD 토큰, 바깥쪽 layout 은 Tailwind 으로 명확히 구분.

# 산출물

- `package.json` — 위 의존성 모두
- `tailwind.config.js` — brand-master / brand-high / brand-middle 토큰
- `vite.config.ts` — react plugin + path alias `@/*` → `src/*`
- `tsconfig.json` — strict + path alias

# 회수

- 번들 사이즈 1.4MB (gz 380KB) — admin + master + student 양 role 모두 단일 chunk. 후속 lazy split 필요 — 백엔드 연동 후 진행 예정
- `S3 / Supabase` 표현이 docstring / CHANGELOG 에 누적 → 백엔드 API 로 일괄 정정 작업 별도 필요

# 참고

- [[../bkurs-fe|프로젝트 hub]]
- [[../task-logs/task-log-s2-series-summary|S2 시리즈 task log]]
