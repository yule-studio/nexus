---
title: bkurs-fe 18 도메인 client 완성
kind: project
status: done
created_at: 2026-05-21
tags: [bkurs-fe, api-client, react-query]
related: [architecture-api-foundation, decision-hook-convention]
home_hub: bkurs-fe
---

## 핵심 요약

bkurs-fe `src/lib/api/*` + `src/lib/hooks/*` 에 18 도메인의 typed
client + react-query hook 추가 완료. 모든 BE envelope endpoint 가 FE 에서
type-safe 하게 호출 가능한 상태.

| 도메인 | client | hook | BE PR |
| --- | --- | --- | --- |
| concept | concept.ts | useConceptDiagnosis | #93 |
| suhaeng | suhaeng.ts | useSuhaeng | #117 |
| saenggibu | saenggibu.ts | useSaenggibu | #119 |
| homework | homework.ts | useHomework | (FE #71) |
| question | question.ts | useQuestion | (FE #71) |
| discussion | discussion.ts | useDiscussion | (FE #73) |
| reading | reading.ts | useReading | (FE #73) |
| interview | interview.ts | useInterview | (FE #73) |
| simulation | simulation.ts | useSimulation | (FE #73) |
| record | record.ts | useRecord | #125 |
| statement | statement.ts | useStatement | #129 |
| performance | performance.ts | usePerformance | #127 |
| dashboard | dashboard.ts | useDashboard | #105 |
| stats | stats.ts | useDashboard | #141 |
| search | search.ts | useDashboard | #133 |
| academy | academy.ts | useAcademy | #143 |
| plan | plan.ts | usePlan | #131 |
| announcement | announcement.ts | useAnnouncement | #135 |

총 **150+ endpoint** typed client, **120+ react-query hook**.

## 내 해석

이전까지 화면 단위로만 hook 을 추가했더니 새 화면을 만들 때마다
boilerplate 가 반복됐다. 18 도메인 client 를 한 곳에 모아두고 모든
endpoint 를 미리 wrap 해두니, 새 화면은 hook 호출만 조립하면 된다.

특히 PR #69 SuhaengManage 가 inroad 의 700 줄 supabase 코드를 341 줄로
줄인 게 결정적 증거 — `useAcademySuhaengList` + `useCreateSuhaengForStudent`
만으로 supabase select/insert 가 사라졌다.

## 적용 맥락

- 새 inroad-high 화면 작업 시 **이 18 도메인 hook 만 import** 하면 끝.
- supabase select/insert 가 보이면 해당 도메인 hook 으로 swap.
- 새 BE endpoint 가 추가되면 동일 file 에 method + hook 추가 — 같은
  파일을 1차 진입점으로.

## envelope alias 정책

같은 이름 (`SubmitRequest` / `GradeRequest` / `FeedbackResponse`) 이
여러 도메인에서 겹쳐 `index.ts` 에서는 `Performance*` / `Statement*`
prefix alias 로 re-export. **import 는 도메인 file 에서 직접** 하는
것이 1순위, alias 는 cross-domain 조립 시에만.

## 관련 노트

- [[architecture-api-foundation]]
- [[decision-hook-convention]]
- [[envelope-migration-progress]] (BE 측 완성)

## 참고

- PR #71 #73 #75 #77 — 4 batch 로 분리
- typecheck PASS, 0 회귀
