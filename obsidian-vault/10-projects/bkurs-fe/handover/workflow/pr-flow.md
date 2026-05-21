---
title: "bkurs-fe — PR 플로우 / git-flow"
kind: handover
status: stable
created_at: 2026-05-22T01:18:00+09:00
tags: [bkurs-fe, handover, workflow, git-flow]
related:
  - "[[../../bkurs-fe]]"
  - "[[../../bkurs-be/handover/workflow/stacked-pr]]"
home_hub: "[[../../bkurs-fe]]"
---

# bkurs-fe — PR 플로우 / git-flow

> BE 의 [[../../bkurs-be/handover/workflow/stacked-pr]] 와 **동일 컨벤션**. 본 문서는 FE 특이점만.
> 기본 git-flow / commit / label / assignee 규칙은 BE 문서 참조.

## 1. 브랜치 / PR 컨벤션 (BE 와 동일)

| 항목 | 규칙 |
| --- | --- |
| 브랜치 | `feature/issue-{n}-{slug}` / `chore/issue-{n}-{slug}` |
| base | dev (보통) — long stack 시 이전 feature branch base 가능 |
| PR 1개 | 1 issue + 최소 3 commit + label + assignee=codwithyc |
| commit | 한국어 + gitmoji + 3 섹션 (변경 이유 / 주요 변경 / 비고) |
| author | codwithyc 통일 |
| auto-merge | 금지 ([[../../../memory/feedback_no_auto_merge]]) |

## 2. FE 특이점

### 2.1 typecheck `--force` 필수 (PR #130)

`tsc -b` 의 incremental cache 가 stale 일 때 변경 못 감지 — `--force` 로 강제 full rebuild.

```bash
npm run typecheck   # = tsc -b --noEmit --force
```

PR CI 도 `npm run typecheck` 호출 — 빌드 통과해도 typecheck 실패면 reject.

### 2.2 verbatimModuleSyntax 위반 빈도

TS 6 의 `verbatimModuleSyntax: true` — type 만 쓰는 import 는 `import type` 강제. 위반 시:

```
error TS1444: 'X' is a type and must be imported using a type-only import
```

해결:

```ts
// ❌
import { Foo, FooBar } from './foo'   // FooBar 가 type 만

// ✅
import { Foo } from './foo'
import type { FooBar } from './foo'
```

### 2.3 alias 동기화

`tsconfig.app.json paths.@/*` ↔ `vite.config.ts alias.@` — 둘 중 한 곳만 갱신하면 빌드 OK 인데
typecheck 실패 (또는 반대). 항상 함께.

### 2.4 supabase 표현 잔재 정리

PR 시 `grep -r "supabase\|Supabase" src/` 로 신규 코드의 잔재 확인. 거의 다 제거되었으나 일부 mock
화면 (concept 등) 에 남아있음 — swap 완료 시 제거.

### 2.5 라우트 추가 시 router.tsx 의 거대 import 블록

`src/router.tsx` 의 상단 60+ named import — 라우트 추가 시 한 줄 추가 + children 배열 한 줄 추가.
lazy split 도입 후 분리 예정.

## 3. 사이클 / Phase (FE 측)

| 사이클 | 내용 |
| --- | --- |
| S0 | setup (#3) — Vite + React 19 + TS + Tailwind + AntD + ESLint + GA |
| S2A~S2N (28 PR) | master 9 + admin 8 + admin 학생상세 19 + high 13 + middle 15 페이지 mock 완성 |
| S1 (#60) | architecture-api-foundation — ApiEnvelope / apiClient / conceptApi |
| (후속) | mock → 실 hook swap 진행 (concept / book 등 잔여) |
| #126~#137 | typecheck 회귀 6 fix + tsBuildInfoFile `.cache/` 이전 + master Questions activation |
| 2026-05-22 | vault 인수인계 재구조 (handover/ 카테고리화) |

자세히 → [[../../bkurs-fe-log]]

## 4. CI 흐름 (`.github/workflows/ci.yml`)

```
on: push branches: [main, dev, release/**]
on: pull_request

jobs:
  quality:
    - actions/checkout
    - actions/setup-node@20 + cache: npm
    - npm ci
    - npm run lint
    - npm run typecheck        # ← --force (PR #130)
    - npm run build
```

`concurrency: cancel-in-progress: true` — 같은 PR 의 새 push 시 이전 run 취소.

## 5. 회수 / pitfall

| 회수 | PR |
| --- | --- |
| typecheck cache stale → false GREEN | #130 (`--force` 추가) |
| `tsBuildInfoFile` 가 node_modules 안 → CI cache 충돌 | #130 (`.cache/tsbuildinfo/` 이전) |
| client.ts 의 axios import 가 type-only 가 일부 누락 | #126 |
| MasterQuestions named import 깨짐 | #128 |

## 6. 관련

- [[../../bkurs-be/handover/workflow/stacked-pr]] — BE 측 git-flow + Phase C cascade (FE 와 공통 정책 SSoT)
- [[../../../memory/feedback_commit_format]] / [[../../../memory/feedback_commit_splitting_policy]] / [[../../../memory/feedback_pr_issue_labels_assignees]]
- [[../../../memory/feedback_no_auto_merge]]
