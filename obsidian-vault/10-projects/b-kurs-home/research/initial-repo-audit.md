---
project: b-kurs-home
type: research
status: done
created_at: 2026-06-02
updated: 2026-06-02
tags: [project, b-kurs-home, audit, research]
related:
  - "[[../b-kurs-home]]"
  - "[[../decisions/decision-no-env-commit-use-example]]"
---

# b-kurs-home 초기 진단

## 확인한 규칙 문서

- commit 규칙: `policies/reference/COMMIT_CONVENTION.md` (gitmoji 1개 + 한국어 제목 + 3 섹션: 변경 이유 / 주요 변경 사항 / 비고, 마침표 X, Co-Authored-By 금지). 첫 커밋만 `:tada: initial commit` — b-kurs-home 은 이미 "Initial commit" 존재 → 일반 gitmoji 사용.
- branch 규칙: `policies/reference/BRANCH_STRATEGY.md` (main 기반, `<prefix>/<desc>`; prefix = feature/refactor/fix/chore/test/docs/hotfix). 거버넌스 hard rail: issue 번호 anchor.
- issue 규칙: 같은 org `bkurs-fe/.github/ISSUE_TEMPLATE/*` (feature/bug/chore + config.yml) 구조 차용. 모든 issue 에 `--assignee codwithyc` + label (사용자 정책).
- PR 규칙: PR 당 최소 3 commit 논리 분할, 템플릿 (관련 이슈 / 과제 내용 / 체크리스트). 자동 머지 금지 → **draft PR 까지만** (사용자 정책).
- vault 기록 규칙: `10-projects/{repo}/` 누적. task-log 파일명은 **날짜 prefix 없이** `task-log-<topic>-issue-<n>.md` (날짜는 frontmatter created_at). 본 작업 지시서는 `YYYY-MM-DD-` 형식을 요청했으나, 지시서 자체의 "기존 규칙을 따르라" 원칙 + 기존 vault 컨벤션을 우선해 날짜 prefix 를 뺐다.

## 원본 레포 상태 (camshf-crypto/b-kurs)

- 구조: Vite 단일 SPA. `index.html` + `src/` (App/main, components 3, pages 18, lib/supabaseContact, assets) + `public/` (favicon/icons/og-image).
- 기술 스택: React 19 + TS 6 + Vite 8 + Tailwind 3 + react-router-dom 7 + @supabase/supabase-js. 배포 Vercel (`vercel.json` SPA rewrite).
- 설정 문제:
  - `package.json` name 이 `inroad-b2b` (레포명과 불일치).
  - `typecheck` / `format` 스크립트 없음 (build 가 `tsc -b && vite build` 로 타입체크 겸함).
  - `.nvmrc` 없음 → CI node 버전 고정 불가.
- 코드 문제: 치명적 문제는 미발견 (정적 마케팅 사이트). 상세 코드 리뷰는 후속 PR.
- 문서 문제: `README.md` 가 Vite 기본 템플릿 README (실 프로젝트 설명 0). AGENTS/CLAUDE/docs 없음.
- **보안/환경변수 문제 (중요)**: `.env` 가 **public 레포에 그대로 커밋**돼 있음.
  - `VITE_BKURS_SUPABASE_URL`, `VITE_BKURS_SUPABASE_ANON_KEY` (publishable anon key).
  - `VITE_` prefix = 빌드시 클라이언트 번들에 노출되는 값이라 "비밀"은 아니나, 실 값을 레포에 커밋하는 건 베스트프랙티스 위반. → 이관 시 `.env.example` 로 대체, 실 `.env` 비커밋. ([[../decisions/decision-no-env-commit-use-example]])
  - `.gitignore` 에 `.env` 명시 없음 (`*.local` 만 있음) → 보강 필요.
- 테스트/빌드 문제: 테스트 없음. CI 없음. 빌드는 표준 vite.

## b-kurs-home 현재 상태 (정리 대상)

- 구조: stub `README.md` (`# bkurs-home`) 한 줄뿐.
- remote: `origin = https://github.com/seum-running/b-kurs-home.git` (private).
- branch: `main` (default), 커밋 1개 (`784f7b9 Initial commit`).
- 기존 파일: `README.md` 만.
- 충돌 가능성: 낮음. 빈 레포라 이관 파일과 충돌 없음 (README 는 덮어씀).

## 이관/정리 전략

- **그대로 옮길 것**: `index.html`, `src/**`, `public/**`, `vite.config.ts`, `eslint.config.js`, `tailwind.config.js`, `postcss.config.js`, `tsconfig*.json`, `vercel.json`.
- **정리해서 옮길 것**: `package.json` (name 수정 + scripts 추가), `.gitignore` (`.env` 제외 보강), `README.md` (한국어 실 문서로 재작성).
- **버릴 것 (이관 X)**: 원본 `.env` (실 값). `.env.example` 로 대체.
- **나중 PR 로 미룰 것**: 페이지 코드 품질 리뷰, 접근성, OG/SEO 메타, 실제 테스트(Playwright/vitest) 도입, prettier 전면 적용 후 format 정리.

## 리스크

- (R1) 원본 anon key 가 이미 public 노출 — 가능하면 Supabase 측에서 키 rotate 권장 (본 PR 범위 밖, decision 에 기록).
- (R2) prettier 신규 도입 시 대량 포맷 diff 발생 가능 → 1차 PR 에서는 format:check 를 CI 에 강제하지 않거나, 도입 시 포맷 커밋을 분리.
- (R3) node/엔진 버전 미고정 → `.nvmrc` 추가로 완화.
- (R4) b-kurs-home 은 main 단일 브랜치, bkurs-fe 는 dev/main 분리 — 본 레포는 BRANCH_STRATEGY SSoT (main 기반) 를 따른다.

## 추천 작업 순서

1. (완료) 규칙 확인 → Issue #1 → 브랜치 `feature/migrate-home-site-issue-1` → vault 초기 문서.
2. config 정리 커밋 (🔧).
3. 소스 이관 커밋 (✨).
4. 문서 (README/AGENTS/CLAUDE/docs) 커밋 (📝).
5. .github (CI/template/dependabot) 커밋 (👷).
6. 로컬 install + lint + typecheck + build 검증.
7. push → draft PR (자동 머지 금지) → task-log 마무리.
