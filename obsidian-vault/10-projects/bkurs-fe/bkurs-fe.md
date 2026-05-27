---
title: "bkurs-fe — B-KURS 프론트엔드 프로젝트 hub"
kind: project
project: bkurs-fe
agent: engineering-agent/tech-lead
status: in-progress
created_at: 2026-05-20T00:15:00+09:00
tags: [project, frontend, react, vite, bkurs]
home_hub: projects
related:
  - "[[../projects]]"
  - "[[../bkurs-be/bkurs-be]]"
  - "[[../../20-areas/frontend/javascript/javascript]]"
---

# bkurs-fe — B-KURS 프론트엔드 프로젝트 hub

> 🚀 **여기서 시작**: bkurs 프로젝트 전체 진입점은 [[../bkurs-be/getting-started]] 에 있다
> (BE + FE 공용). 화면 작업만 한다면:
>
> - **[[screen-logic-and-api-map]]** — 52 화면 × hook × endpoint 매핑 + 신규 화면 체크리스트
> - **[[architecture-api-foundation]]** — ApiEnvelope 구조 / apiClient
> - **[[fe-domain-clients-complete]]** — 18 도메인 client 목록
> - **[[decisions/decision-hook-convention]]** — `lib/hooks/` 컨벤션 ADR

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-20 | engineering-agent/tech-lead | 사후 회고 hub — S2 시리즈 완성 후 시점에서 정리 |

**[[../projects|↑ 10-projects]]** · **짝 프로젝트 [[../bkurs-be/bkurs-be]]**

---

## 1. 프로젝트 정의

inroad-high (camshf-crypto 의 운영 React 코드) 를 디자인 동일하게 유지하면서 **senior 분해 + Vite 8 / React 19 / TS 6 / Tailwind / Ant Design 6 / Jotai / TanStack Query 기반으로 재구성**.

- Repo: https://github.com/seum-running/bkurs-fe
- 모든 데이터 mock — 백엔드 API 는 [[../bkurs-be/bkurs-be]] 에서 점진 도입

## 2. 완성도 (2026-05-20 기준)

| 영역 | 페이지 수 |
|---|---|
| master role | 9/9 ✅ |
| admin role (보조 페이지) | 8/8 ✅ |
| admin 학생상세 (고등 10 + 중등 9) | 19/19 ✅ |
| 학생 부트스트랩 (login/signup/connect/pending × 2 role) | ✅ |
| high-student 도메인 | 9/9 ✅ |
| middle-student 도메인 | 11/11 ✅ |

PR 28개 (S2A → S2N) 모두 머지, issue 모두 close.

## 3. 핵심 senior 패턴

- **AiAnalysisPanel 6회 재사용** — high/middle ExpectTab + PastTab + middle PresentationTab + SuhaengTab
- **ThreadView 7회 재사용** — 5 단계 답변 thread (high-student 3 + middle-student 4)
- **ScoreRadar 2회 공유** — middle Past + Presentation
- **타입 (AIAnalysisResult / ScoreItem / StudentAnswerThread) 중복 정의 0** — import 재사용
- 화면 우선 (admin 학생상세) vs 깊이 분해 (S2E 시리즈) 정책을 PR 단위로 명시

## 4. 회고

- **vault 기록 빼먹음** — 28 PR 진행 동안 본 노트 / 20-areas 보강을 단 한 번도 안 함. 사용자가 2026-05-20 에 명시적으로 지적 → [[../../memory/feedback_vault_project_tracking|정책 memory]] 신설 + 본 hub 사후 작성
- **숨겨진 가정 — 백엔드 = Supabase**: docstring / CHANGELOG 에 "S3 단계 Supabase 연동" 표현이 누적됨. 2026-05-19 사용자가 "supabase 아니고 백엔드 API" 라고 정정 → 표현 일괄 정정 작업이 추가 필요 (TODO)
- **번들 사이즈**: 1.4MB / gz 380KB — admin + master + 양 role student 모두 단일 chunk. lazy split 도입은 백엔드 연동 후

## 5. 다음 작업 후보

1. **백엔드 API 연동** — [[../bkurs-be/bkurs-be]] 진행 따라 mock → axios hook 으로 한 줄씩 교체
   - **S1 DONE (#60)**: [[architecture-api-foundation|Supabase → bkurs-be API 전환 기반]] — envelope / apiClient / conceptApi
   - S2/S3 후속: 화면별 hook 격리 + 실 호출 swap
2. Supabase wrapper 제거 + docstring 표현 일괄 정정
3. 번들 lazy split (route-level)
4. unit test (Vitest) / E2E (Playwright)
5. i18n / 다크모드

## 6. cross-link

- [[../bkurs-be/bkurs-be]] — 짝 백엔드 프로젝트
- [[../../memory/feedback_vault_project_tracking|vault project tracking 정책]]
- [[../../memory/feedback_vault_doc_style|vault 가이드라인 문서 스타일]]
