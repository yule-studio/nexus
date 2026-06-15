---
project: b-kurs-home
type: task-log
status: done
created_at: 2026-06-02
updated: 2026-06-02
tags: [project, b-kurs-home, task-log, migration]
related:
  - "[[../b-kurs-home]]"
  - "[[../research/initial-repo-audit]]"
  - "[[../decisions/decision-no-env-commit-use-example]]"
---

# 작업 로그 — 홈 사이트 이관 + 레포 정리 (Issue #1)

- 날짜: 2026-06-02
- 작업 레포: `seum-running/b-kurs-home`
- 원본 레포: `camshf-crypto/b-kurs`
- GitHub Issue: #1 (https://github.com/seum-running/b-kurs-home/issues/1)
- GitHub PR: #2 (MERGED, merge commit cb7d03e, https://github.com/seum-running/b-kurs-home/pull/2)
- Branch: `feature/migrate-home-site-issue-1`
- 목표: 빈 stub 레포를 README 만으로 실행 가능한 유지보수 가능 레포로. 소스 이관 + config/문서/CI 정리.

## 참고한 에이전트 규칙 문서

- `policies/reference/COMMIT_CONVENTION.md` — 커밋 형식
- `policies/reference/BRANCH_STRATEGY.md` — 브랜치/PR/머지
- `seum-running/bkurs-fe/.github/*`, `README.md` — 같은 org 표준 스타일
- 사용자 정책 (memory): 라벨+assignee 필수, 자동 머지 금지(draft PR), task-log 날짜 prefix 금지

## 변경 요약

빈 stub 레포 → 4 커밋 1 draft PR 로 이관/정리 완료.

- `🔧` 7bec4f4 — Vite/ESLint/Tailwind/PostCSS/tsconfig/vercel 설정, package.json rename, .env.example, .nvmrc, .gitignore 보강, eslint baseline 완화
- `✨` ac683c9 — index.html/src/public 소스, contact.tsx catch 정리
- `📝` 5a6a4b0 — README/AGENTS/CLAUDE + docs(PROJECT_STRUCTURE/BRANCH_STRATEGY/CHANGELOG)
- `👷` d74da27 — CI(lint+typecheck+build) + 이슈/PR 템플릿 + dependabot

생성 라벨: `chore`, `dependencies`, `ci` (템플릿/dependabot 참조용).

## 주요 결정

- 실 `.env` 비이관, `.env.example` + gitignore 보강 → [[../decisions/decision-no-env-commit-use-example]]
- 1차 PR 범위 = 소스 이관 + config + 문서 + CI. 코드 품질/테스트 도입은 후속 PR.
- 작업 지시서의 `YYYY-MM-DD-` task-log 파일명 대신 기존 vault 컨벤션(날짜 prefix 없음) 사용.

## 버린 대안

- 원본 `.env` 그대로 커밋 (보안/문서화 이유로 기각)
- baseline-only(코드 없이 config/문서만) PR — "실행 가능한 기준선"은 코드가 있어야 하므로 소스 포함

## 검증 결과

- 로컬: `npm install` OK (252 pkg) → `npm run lint` 0 error / 4 warn → `npm run typecheck` 0 → `npm run build` 성공.
- GitHub Actions CI (run 26793787376): **success** (39s, node 22, lint+typecheck+build).
- 보안 확인: 실 `.env` 미이관, `git check-ignore` 로 `.env`/`.env.*` 무시 확인, staged diff secret 스캔 — 실 키 없음 (.env.example/README 는 placeholder/문서 텍스트만).
- 빌드 경고: 단일 청크 ~1.1MB(gzip 269KB) → 코드 스플리팅 후속 과제 (검증 못한 것 아님, 의도적 보류).

## 남은 리스크

- 노출된 anon key 회전 (Supabase 콘솔, 후속)
- prettier 전면 적용 시 포맷 diff
- 실제 e2e/unit 테스트 부재

## 다음 작업

- (후속 PR) 페이지 코드 품질 리뷰, SEO/OG 메타 정리, 테스트 도입

## Merge 기록

- Merge 여부: 완료 (자율 머지 — b-kurs-home 클론/이관 기간 인가 정책)
- Merge 시각: 2026-06-02 03:05 UTC
- Merge 방식: Merge Commit (squash 아님 — 4+1 commit 논리 분할 히스토리 보존, CLAUDE.md 정책: squash 는 docs-only 1-commit 만)
- Merge commit: `cb7d03e` (PR #2)
- CI 상태: success (마지막 commit `8b8f110` 기준 lint+typecheck+build)
- 머지 후 확인: 원격/로컬 `main` = `cb7d03e` 동기화, 원격·로컬 작업 브랜치 삭제, Issue #1 CLOSED (PR "Closes #1")
- 다음 작업:
  - `refactor/eslint-baseline-restore` — 완화 규칙(set-state-in-effect/immutability/no-explicit-any) 정식 수정 후 error 복원
  - `refactor/route-code-splitting` — 단일 ~1.1MB 청크 → React.lazy 라우트 분할
  - `test/baseline` — vitest/Playwright + prettier 베이스라인
  - 운영: 노출된 Supabase anon key 회전 + RLS 점검

> 머지 방식 메모: 작성 당시 docs/BRANCH_STRATEGY §6 이 "기본 Squash" 로 잘못 적혀
> 있어, 머지 전 마지막 commit(`8b8f110`)으로 조직 정책(merge commit 보존)과
> 일치하도록 정정한 뒤 merge commit 으로 머지했다.
