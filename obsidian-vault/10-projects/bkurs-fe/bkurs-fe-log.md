---
title: "bkurs-fe — PR / 사이클 history"
kind: project-log
project: bkurs-fe
status: in-progress
created_at: 2026-05-20T00:15:00+09:00
updated_at: 2026-05-22T01:15:00+09:00
tags: [project-log, frontend, bkurs, history]
home_hub: "[[bkurs-fe]]"
related:
  - "[[bkurs-fe]]"
  - "[[handover/index]]"
---

# bkurs-fe — PR / 사이클 history

> S2 시리즈 완성 후 사후 회고로 시작. 코드 측 룰 / 가이드는 [[handover/index]] 가 SSoT. 본 문서는
> 시간축 history.

**[[bkurs-fe|↑ 프로젝트 hub]]** · **[[handover/index|인수인계 문서 인덱스]]**

## 1. 문서 버전 누적

| 문서 버전 | 작성일 | 주요 변경 사항 |
| --- | --- | --- |
| v.1.0.0 | 2026-05-20 | 사후 회고 hub — S2 시리즈 완성 후 시점에서 정리 |
| v.1.1.0 | 2026-05-21 | 인수인계 문서 (8 핸드오버 doc) 신설 — directory-structure / role-modules / lib-layer-guide / shared-components / routing-and-protection / design-system / state-and-auth / build-and-ci |
| v.1.2.0 | 2026-05-22 | handover/ 카테고리화 (architecture/modules/pages/api-clients/operations/workflow/progress) + 22 API client 도메인 doc + 5 role pages doc + 배포 가이드 + log/index 분리 |

## 2. 완성도 (2026-05-20 기준)

| 영역 | 페이지 수 |
| --- | --- |
| master role | 9/9 ✅ |
| admin role (보조 페이지) | 8/8 ✅ |
| admin 학생상세 (고등 10 + 중등 9) | 19/19 ✅ |
| 학생 부트스트랩 (login/signup/connect/pending × 2 role) | ✅ |
| high-student 도메인 | 13/13 ✅ |
| middle-student 도메인 | 15/15 ✅ |

PR 28개 (S2A → S2N) 모두 머지, issue 모두 close.

## 3. 핵심 senior 패턴 (보존)

- **AiAnalysisPanel 6회 재사용** — high/middle ExpectTab + PastTab + middle PresentationTab + SuhaengTab
- **ThreadView 7회 재사용** — 5 단계 답변 thread (high-student 3 + middle-student 4)
- **ScoreRadar 2회 공유** — middle Past + Presentation
- **타입 (AIAnalysisResult / ScoreItem / StudentAnswerThread) 중복 정의 0** — import 재사용
- 화면 우선 (admin 학생상세) vs 깊이 분해 (S2E 시리즈) 정책을 PR 단위로 명시

자세히 → [[handover/modules/shared-components]]

## 4. 회고

- **vault 기록 빼먹음** — 28 PR 진행 동안 본 노트 / 20-areas 보강을 단 한 번도 안 함. 사용자가 2026-05-20 에 명시적으로 지적 → [[../../memory/feedback_vault_project_tracking|정책 memory]] 신설 + 본 hub 사후 작성
- **숨겨진 가정 — 백엔드 = Supabase**: docstring / CHANGELOG 에 "S3 단계 Supabase 연동" 표현이 누적됨. 2026-05-19 사용자가 "supabase 아니고 백엔드 API" 라고 정정 → 표현 일괄 정정 작업 진행 (대부분 완료, 잔여 mock 화면 swap 후 정리)
- **번들 사이즈**: 1.4MB / gz 380KB — admin + master + 양 role student 모두 단일 chunk. lazy split 도입은 BE 연동 후
- **typecheck 회귀 누적** — TS 6.0 / verbatimModuleSyntax 도입 시 82 회귀. 6 fix PR + tsBuildInfoFile `.cache/` 이전 (#130) 으로 해결. 자세히 → [[handover/progress/regression-2026-05-21]]

## 5. 다음 작업 후보

1. **백엔드 API 연동 잔여** — [[handover/progress/screen-api-map]] 진척표 따라 mock → hook swap
   - S1 DONE (#60): [[handover/architecture/api-foundation|Supabase → bkurs-be API 전환 기반]]
   - S2~S3 후속: 화면별 hook 격리 + 실 호출 swap 완료
2. **Supabase wrapper 완전 제거** + docstring 표현 일괄 정정
3. **번들 lazy split** (route-level `React.lazy()` + `Suspense`)
4. **unit test** (Vitest) / E2E (Playwright)
5. **i18n / 다크모드**
6. **lib/auth/atoms.ts legacy shim 제거** (concept 마지막 swap 후)

## 6. PR 누적 (선별)

| 일자 | PR | 내용 |
| --- | --- | --- |
| 2026-05-18 | #3 | Vite + React 19 + TS / Tailwind + AntD / ESLint + Prettier / src 스켈레톤 + Router/Provider/Supabase / GitHub Actions CI / 컨벤션 docs + ISSUE·PR 템플릿 |
| 2026-05-19~20 | S2A~S2N (28 PR) | master 9 + admin 8 + admin 학생 상세 19 + high-student 13 + middle-student 15 페이지 |
| 2026-05-21 | #60 | architecture-api-foundation — ApiEnvelope / apiClient / conceptApi |
| 2026-05-21 | #126 | client.ts type-only import 분리 (verbatimModuleSyntax) |
| 2026-05-21 | #128 | MasterQuestions named → default import 정정 |
| 2026-05-21 | #130 | CI typecheck `--force` + tsBuildInfoFile `.cache/` 외부 |
| 2026-05-21 | #131 | typecheck Type check 주석 |
| 2026-05-21 | #132~#137 | master/Questions placeholder → PastQuestion / MajorQuestion CRUD 활성화 + 학생 측 swap |
| 2026-05-21 | #149~#151 | (BE 측 매핑) change-password / grade 1~6 / student membership view |
| 2026-05-22 | vault 인수인계 재구조 | 핸드오버 8 doc → handover/ 카테고리화 + 5 role pages + 22 api-clients sub doc + log/index/deployment 분리 |

## cross-link

- [[bkurs-fe]] — 프로젝트 hub (간소화된 정의 + 결정)
- [[handover/index]] — 인수인계 문서 전체 인덱스
- [[handover/pages/catalog]] — 4 role 페이지 인덱스
- [[handover/api-clients/catalog]] — 22 API client 인덱스
- [[handover/progress/screen-api-map]] — 60+ 화면 swap 진척
- [[handover/progress/regression-2026-05-21]] — typecheck 회귀 분석
