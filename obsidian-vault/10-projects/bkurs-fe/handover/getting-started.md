---
title: "bkurs-fe — getting started (역할별 진입)"
kind: handover
status: active
created_at: 2026-05-22T01:15:00+09:00
tags: [bkurs-fe, handover, onboarding]
home_hub: "[[../bkurs-fe]]"
related:
  - "[[../bkurs-fe]]"
  - "[[index]]"
---

# bkurs-fe — getting started

> FE 신규 합류자 5분 진입. 역할별 read path. 전체 인덱스는 [[index]].

## 0. 30초 셋업

```bash
cd bkurs-fe
nvm use            # Node 20
cp .env.example .env.local
# VITE_API_BASE_URL=http://localhost:8080/api 편집
npm install
npm run dev        # http://localhost:5173
```

BE 도 띄워야 실 데이터 — [[../../bkurs-be/handover/getting-started]].

## 1. 역할별 read path

### 처음 합류
1. [[index]] — 전체 인수인계 인덱스
2. [[architecture/directory-structure]] — src/ 전체 지도
3. [[../../bkurs-be/handover/getting-started]] — BE 측 진입

### 화면 추가 / 수정
1. [[pages/catalog]] — 4 role 페이지 인덱스
2. [[architecture/routing-and-protection]] — router + 학생 4 status 가드
3. [[modules/shared-components]] — ThreadView / AiAnalysisPanel 재사용
4. [[modules/design-system]] — brand 토큰 (raw hex 금지)

### API 호출 swap
1. [[api-clients/catalog]] — 22 client 인덱스 + BE 매핑
2. [[modules/lib-layer-guide]] — axios + react-query 컨벤션
3. [[progress/screen-api-map]] — swap 진척표 (DONE/MOCK/N/A)
4. [[decisions/decision-hook-convention]] — hook 위치 / query key

### 배포 / CI
1. [[operations/deployment]] — Cloudflare Pages 또는 S3+CloudFront
2. [[modules/build-and-ci]] — Vite/tsc/tsBuildInfoFile 함정 + GA
3. [[workflow/pr-flow]] — git-flow + commit 컨벤션

### 인증 / 가드
1. [[architecture/routing-and-protection]] — 4 role layout + 학생 4 status 분기
2. [[architecture/state-and-auth]] — Jotai atom + 401 흐름

## 2. "이것만은"

- **secret 금지 commit**: `.env.local` (gitignore), `VITE_*` 에 secret 금지 (JS 번들 노출)
- **디자인 토큰만**: raw hex 직접 사용 금지 — brand-high/middle/master + ink/line ([[modules/design-system]])
- **role import 금지**: `pages/high-student/**` 가 `pages/middle-student/**` import 금지 (cross-role 은 `_shared/` 또는 `lib/`)
- **typecheck 함정**: TS 6 verbatimModuleSyntax — type-only import 분리. `tsBuildInfoFile` 은 `.cache/` 외부 ([[progress/regression-2026-05-21]])
- **supabase 점진 제거**: 새 코드는 axios + lib/api 만

## 3. 다음에 볼 것

- 전체 → [[index]]
- BE 매핑 → [[../../bkurs-be/handover/index]]
- 화면 swap roadmap → [[progress/screen-api-map]]
- 사이클 회고 → [[../bkurs-fe-log]]

## 4. 관련

- [[../bkurs-fe]] — 프로젝트 hub
- [[../bkurs-fe-log]] — PR / 사이클 history
- [[../../bkurs-be/handover/getting-started]] — BE 측 진입
