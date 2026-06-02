---
project: b-kurs-home
type: project-overview
status: active
created_at: 2026-06-02
updated: 2026-06-02
tags: [project, bkurs, b-kurs-home, marketing-site, fe, vite, react]
related:
  - "[[research/initial-repo-audit]]"
  - "[[task-logs/task-log-home-site-migration-issue-1]]"
  - "[[decisions/decision-no-env-commit-use-example]]"
  - "[[bkurs/01-project-map]]"
home_hub: "[[projects]]"
---

# b-kurs-home

## 1. 한 줄 요약

`seum-running/b-kurs-home` — 비커스 (BKURS) 의 **마케팅 / 홈 (랜딩) 사이트**.
원본 `camshf-crypto/b-kurs` (패키지명 `inroad-b2b`) 를 이관/정리해서
유지보수 가능한 레포로 만든다.

## 2. 레포 좌표

| 항목 | 값 |
| --- | --- |
| 정리 대상 | `https://github.com/seum-running/b-kurs-home` (private) |
| 원본 참고 | `https://github.com/camshf-crypto/b-kurs` (public, TypeScript) |
| 로컬 작업 | `/Users/masterway/masterway-dev/bkurs/b-kurs-home` |
| 기본 브랜치 | `main` |

> 주의: 이 레포는 **앱 본체가 아니다**. 앱 본체 FE/BE 는
> [[bkurs/01-project-map]] 의 `bkurs-fe` (`seum-running/bkurs-fe`) /
> `bkurs-be` (`seum-running/bkurs-be`). `b-kurs-home` 은 공개 홈/랜딩 사이트.

## 3. 기술 스택 (원본 기준)

| 영역 | 기술 |
| --- | --- |
| 빌드 | Vite 8 + React 19 + TypeScript 6 |
| 스타일 | Tailwind CSS 3 + PostCSS + autoprefixer |
| 라우팅 | react-router-dom 7 |
| BaaS | Supabase (contact form 만 사용) |
| 배포 | Vercel (SPA rewrite) |
| 품질 | ESLint 9 (flat config) |

페이지 구성: landing / why / pricing / partners / login-select / contact /
guide(home·category·article) / service(case·effect·product) / terms(terms·privacy·refund).

## 4. 진행 현황

- **2026-06-02** — Issue #1, 이관/정리 1차 PR 진행 ([[task-logs/task-log-home-site-migration-issue-1]]).
  - 빈 stub 레포 → 소스 이관 + config 정리 + README/AGENTS/CLAUDE + docs + .github(CI/template/dependabot).

## 5. 작업 규칙 (참조 — invent 금지)

- 커밋/브랜치/PR 규칙은 에이전트 레포 SSoT 를 그대로 따른다:
  `policies/reference/COMMIT_CONVENTION.md`, `policies/reference/BRANCH_STRATEGY.md`.
- 같은 org 표준 템플릿/CI/README 스타일은 `seum-running/bkurs-fe` 를 참조.
- 결정은 `decisions/` 에 ADR 로 남긴다.

## 6. 관련 노트

- [[research/initial-repo-audit]] — 초기 진단
- [[decisions/decision-no-env-commit-use-example]] — .env 비커밋 결정
- [[task-logs/task-log-home-site-migration-issue-1]] — 1차 이관 작업 로그
- [[bkurs/01-project-map]] — 앱 본체 (FE/BE) 큰 그림
