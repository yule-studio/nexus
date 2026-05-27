---
title: "bkurs-fe — B-KURS 프론트엔드 프로젝트 hub"
kind: project
project: bkurs-fe
agent: engineering-agent/tech-lead
status: in-progress
created_at: 2026-05-20T00:15:00+09:00
updated_at: 2026-05-22T01:15:00+09:00
tags: [project, frontend, react, vite, bkurs]
home_hub: projects
related:
  - "[[../projects]]"
  - "[[../bkurs-be/bkurs-be]]"
  - "[[bkurs-fe-log]]"
  - "[[handover/index]]"
  - "[[../../20-areas/frontend/javascript/javascript]]"
---

# bkurs-fe — B-KURS 프론트엔드 프로젝트 hub

> 본 hub 는 **프로젝트 정의 + 기술 결정 + 운영 원칙** 만. 코드 인수인계는 **[[handover/index]]**,
> PR / 사이클 history 는 **[[bkurs-fe-log]]**.

**[[../projects|↑ 10-projects]]** · **짝 프로젝트 [[../bkurs-be/bkurs-be]]**

## 진입

| 목적 | 문서 |
| --- | --- |
| 코드 인수인계 (전체 인덱스) | [[handover/index]] |
| 4 role 페이지 | [[handover/pages/catalog]] |
| 22 API client | [[handover/api-clients/catalog]] |
| PR / 사이클 history | [[bkurs-fe-log]] |
| 화면 swap 진척 | [[handover/progress/screen-api-map]] |
| 빠른 진입 (FE) | [[handover/index]] §0 |

---

## 1. 프로젝트 정의

inroad-high (camshf-crypto 의 운영 React 코드) 의 **디자인 동일하게 유지** + senior 분해 +
Vite 8 / React 19 / TS 6 / Tailwind / Ant Design 6 / Jotai / TanStack Query 기반 재구성.

- Repo: https://github.com/seum-running/bkurs-fe
- BE 매핑: [[../bkurs-be/bkurs-be]] (mock → 실 API 점진 swap, 거의 완료)

## 2. 기술 결정

| 영역 | 선택 | 이유 |
| --- | --- | --- |
| 빌드 | Vite 8 | 빠른 HMR + ES modules |
| 프레임워크 | React 19 | 사용자 결정 + concurrent features |
| 언어 | TypeScript 6 strict + verbatimModuleSyntax | type-only import 강제 + erasableSyntaxOnly |
| 스타일 | Tailwind 3.4 + AntD 6 | inroad-high 디자인 토큰 보존 + AntD ThemeConfig 일치 |
| 상태 | Jotai 2 (atomWithStorage) | localStorage sync + atomic / minimal |
| 서버 캐시 | TanStack Query 5 | invalidate / optimistic update / DevTools |
| 라우팅 | React Router 7 | createBrowserRouter API + lazy 후속 |
| HTTP | axios 1 + interceptor | Bearer 자동 + 401 처리 |
| 품질 | ESLint 9 flat + Prettier 3 | strict React hooks + jsx-runtime |
| BaaS (legacy) | Supabase | mock 단계 잔재 — 점진 제거 |

자세히 → [[handover/decisions/decision-vite-react-19-tailwind-antd-jotai]]

## 3. 로컬 개발

```bash
cd bkurs-fe
cp .env.example .env.local
vi .env.local      # VITE_API_BASE_URL 만 신경

npm install
npm run dev        # → http://localhost:5173

npm run typecheck  # tsc -b --noEmit --force
npm run build      # tsc -b && vite build → dist/
npm run lint
```

local 필수 env:
- `VITE_API_BASE_URL=http://localhost:8080/api` (BE local 띄울 때)

선택 (legacy):
- `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` — concept 진단 fallback (점진 제거)

자세히 → [[handover/modules/build-and-ci]]

## 4. 작업 원칙

- **vault 기록**: PR 별 본 노트 + 20-areas 보강 — [[../../memory/feedback_vault_project_tracking]]
- **commit 규칙**: 한국어 + gitmoji + 3 섹션
- **PR 당 최소 3 commit** + label + assignee — [[../../memory/feedback_pr_issue_labels_assignees]]
- **자동 머지 금지** — [[../../memory/feedback_no_auto_merge]]
- **author**: codwithyc 통일
- **secret 금지 commit**: `.env.local` (gitignore), VITE_* 에 secret 절대 금지

자세히 → [[handover/workflow/pr-flow]] (BE 와 공통 정책)

## 5. 현재 상태 (요약)

| 항목 | 값 |
| --- | --- |
| dev 브랜치 | dev — S2 시리즈 28 PR 머지 완료 + 후속 fix #126~#137 / #149~#151 |
| 페이지 | 60+ (master 9 / admin 27 / high 17 / middle 19) |
| API client | 22 도메인 (lib/api/) |
| Hook | 18 도메인 (lib/hooks/) — useSearch / useStats 등 후속 |
| 번들 | 1.4MB / gz 380KB (lazy split 미구현) |

상세 진행 history → [[bkurs-fe-log]]

## 6. cross-link

- [[handover/index]] — 인수인계 문서 전체 인덱스 (필독)
- [[bkurs-fe-log]] — PR / 사이클 진행 로그
- [[../bkurs-be/bkurs-be]] — 짝 백엔드 hub
- [[../bkurs-be/handover/domains/catalog]] — BE 도메인 ↔ FE api-clients 매핑
- [[../../20-areas/frontend/javascript/javascript]] — JS/TS 일반
- [[../../memory/project_bkurs_fe_reimplementation]] — 프로젝트 memory
