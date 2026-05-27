---
title: "bkurs-fe — 인수인계 문서 인덱스"
kind: handover
status: stable
created_at: 2026-05-22T01:00:00+09:00
tags: [bkurs-fe, handover, index]
home_hub: "[[../bkurs-fe]]"
related:
  - "[[../bkurs-fe]]"
  - "[[../bkurs-fe-log]]"
---

# bkurs-fe — 인수인계 문서 인덱스

> FE 신규 합류자 / 화면 작업자의 1순위 진입점. handover/ 안의 가이드 + 4 role 페이지 + 22 API
> client + 운영 가이드 + ADR 디렉토리를 모두 표면화. 프로젝트 정의는 [[../bkurs-fe]], 사이클
> 회고 / regression / PR 누적은 [[../bkurs-fe-log]].

## 0. 빠른 진입

| 역할 | 1순위 |
| --- | --- |
| 처음 합류 | [[getting-started]] (역할별 read path) |
| 화면 코드 작업 | [[architecture/directory-structure]] + [[pages/catalog]] |
| API 호출 추가 / swap | [[modules/lib-layer-guide]] + [[api-clients/catalog]] |
| 인증 / 라우팅 | [[architecture/routing-and-protection]] + [[architecture/state-and-auth]] |
| 디자인 토큰 / AntD | [[modules/design-system]] |
| 공유 컴포넌트 (Panel/Thread/Radar) | [[modules/shared-components]] |
| 빌드 / CI | [[modules/build-and-ci]] |
| 배포 | [[operations/deployment]] |

## 1. 코어 가이드

진입 순서대로:

1. [[architecture/directory-structure]] — `src/` 전체 지도 (pages role / lib / components / styles)
2. [[architecture/api-foundation]] — ApiEnvelope / apiClient / unwrap 도입 (#60 ADR)
3. [[architecture/routing-and-protection]] — `router.tsx` role-based + 인증 가드 + 학생 4 status 분기
4. [[architecture/state-and-auth]] — Jotai atom + supabase 점진 제거 + 4 role 인증 흐름
5. [[modules/lib-layer-guide]] — `lib/api` 25 client + `lib/hooks` 18 hook + query key 컨벤션
6. [[modules/shared-components]] — AiAnalysisPanel / ThreadView / ScoreRadar 재사용 매핑
7. [[modules/design-system]] — Tailwind brand 토큰 + AntD ThemeConfig + inroad-high 1:1
8. [[modules/build-and-ci]] — Vite + TS 6 strict + ESLint flat + GitHub Actions

## 2. 페이지 (role 별)

5 role × 60+ 페이지. 자세히 → [[pages/catalog]]

- [[pages/admin]] — 학원장 (OWNER) — 보조 8 + 학생 상세 19 페이지
- [[pages/master]] — 본사 (MASTER) — 9 페이지
- [[pages/high-student]] — 고등 학생 (STUDENT_HIGH) — 부트스트랩 4 + 학습 13
- [[pages/middle-student]] — 중등 학생 (STUDENT_MIDDLE) — 부트스트랩 4 + 학습 15
- [[pages/_shared]] — role 간 공용 (auth 4 hook)

## 3. API client / hook (22 도메인)

BE 의 22 functional 도메인과 1:1. 자세히 → [[api-clients/catalog]]

- core: [[api-clients/auth]] / [[api-clients/academy]] / [[api-clients/announcement]] / [[api-clients/studentProfile]] / [[api-clients/plan]]
- learning: [[api-clients/homework]] / [[api-clients/statement]] / [[api-clients/question]] / [[api-clients/interview]] / [[api-clients/record]] / [[api-clients/discussion]] / [[api-clients/reading]] / [[api-clients/performance]] / [[api-clients/simulation]]
- aggregator: [[api-clients/search]] / [[api-clients/stats]] / [[api-clients/dashboard]]
- content: [[api-clients/concept]] / [[api-clients/suhaeng]] / [[api-clients/saenggibu]] / [[api-clients/pastQuestion]] / [[api-clients/majorQuestion]]

> file/pdf/notification/ai/stt 는 BE gateway 가 처리 — FE 직접 호출 없음.

## 4. 운영 / 워크플로

- [[operations/deployment]] — FE 배포 (Cloudflare Pages 또는 S3+CloudFront + Cloudflare)
- [[workflow/pr-flow]] — git-flow + commit 컨벤션 (BE 와 공통)

## 5. ADR / 사이클 / 회고

- `decisions/` — ADR (decision-hook-convention / decision-vite-react-19-tailwind-antd-jotai 등)
- `progress/screen-api-map.md` — 60+ 화면 swap 진척표 SSoT
- `progress/regression-2026-05-21.md` — typecheck 회귀 분석
- `progress/retrospectives/` — 사이클 회고
- `task-logs/` — PR 별 작업 로그

## 6. 외부 참고

- [[../../bkurs-be/bkurs-be]] — 짝 백엔드 hub
- [[../../bkurs-be/handover/index]] — BE 인수인계 인덱스
- [[../../bkurs-be/handover/domains/catalog]] — BE 29 도메인 — FE api-clients 와 1:1 매핑
- [[../../../20-areas/frontend/javascript/javascript]] — JS/TS 일반
- [[../../../memory/project_bkurs_fe_reimplementation]] — 프로젝트 memory

## cross-link

- [[../bkurs-fe]] — 프로젝트 hub
- [[../bkurs-fe-log]] — PR / 사이클 history
