---
title: bkurs-fe — 누적 typecheck 회귀 분석 + fix 시리즈 (RESOLVED, 2026-05-21)
kind: report
status: resolved
created_at: 2026-05-21
tags: [bkurs-fe, regression, typecheck, postmortem]
related:
  - "[[bkurs-fe]]"
  - "[[screen-logic-and-api-map]]"
home_hub: "[[bkurs-fe]]"
---

# bkurs-fe 누적 typecheck 회귀 보고서 (RESOLVED, 2026-05-21)

> ✅ **fix 완료**: A~E 5 PR + CI 갱신 1 PR = 총 6 PR 머지. typecheck 82 → 0. 재발 방지 장치 도입.
> 본 노트는 진단 + 시간순 기록 + 재발 방지 가이드.

## 0. fix 시리즈 (시간순)

| # | 카테고리 | PR | errors | 머지 |
|---|----------|----|--------|------|
| 1 | A. BookList shape (high+middle) | [#121](https://github.com/seum-running/bkurs-fe/pull/121) | -24 → 58 | ✅ |
| 2 | B. Homework shape | [#123](https://github.com/seum-running/bkurs-fe/pull/123) | -6 → 52 | ✅ |
| 3 | D. concept atoms shim | [#125](https://github.com/seum-running/bkurs-fe/pull/125) | -7 → 45 | ✅ |
| 4 | E. client.ts type-only | [#127](https://github.com/seum-running/bkurs-fe/pull/127) | -2 → 43 | ✅ |
| 5 | C. master/questions placeholder | [#129](https://github.com/seum-running/bkurs-fe/pull/129) | -33 → 0 | ✅ |
| 6 | CI: --force + tsBuildInfoFile 외부 | [#131](https://github.com/seum-running/bkurs-fe/pull/131) | (재발 방지) | ✅ |

진행 순서는 사용자 지시 (A → B → D → E → C → CI).

## 1. 한 줄 요약

`tsc -b --noEmit` (CI script) 가 incremental `.tsbuildinfo` 캐시 활용. `tsBuildInfoFile`
이 `node_modules/.tmp/` 에 있어 setup-node 의 npm cache 와 함께 stale 보존 → CI 가
누적 회귀 82 errors 를 silent 통과. fix 머지 + 캐시 격리로 0 errors 달성.

## 2. 검증 명령 차이 (정정)

> 초기 진단의 "tsc --noEmit 가 root tsconfig 의 `files: []` 만 검사" 가정은 부정확. 실제로는
> `tsc -b --noEmit` 가 references 모두 검사하지만 **incremental `.tsbuildinfo` 캐시** 가
> stale 인 상태로 PASS 반환한 게 root cause.

| 명령 | 동작 | clean cache | stale cache |
|------|------|-------------|-------------|
| `npx tsc --noEmit` | root tsconfig 만 (`files: []` → 0 file) | exit 0 | exit 0 |
| `npx tsc -b --noEmit` (npm typecheck) | composite, incremental | exit 0 + 82 errors | exit 0 (stale 무시) |
| `npx tsc -b --noEmit --force` | incremental 무시 강제 전수 | exit 0 + 82 errors | exit 0 + 82 errors |
| `npx tsc --noEmit -p tsconfig.app.json` | src 전체 단발 검사 | exit 0 + 82 errors | (캐시 무관) |

> ⚠️ tsc 는 errors 가 있어도 stdout 에 출력하면서 exit code 0 일 때가 있음. CI 의 npm run
> typecheck 는 exit code 만 확인 → PASS 로 보고. **--force 가 필수**.

## 3. 카테고리별 분류 (82 errors)

### 카테고리 A — 내가 만든 회귀: BookList shape 미스매치 (24 errors)

**파일**: `src/pages/high-student/_pages/book/BookList.tsx` (12) +
`src/pages/middle-student/_pages/book/BookList.tsx` (12)

**원인**: PR #83 / #85 (Book 도메인 swap) 작성 시 `ReadingNoteView` shape 를 잘못 가정.

| 코드 가정 | BE 실제 (`src/lib/api/reading.ts`) |
|-----------|------------------------------------|
| `title` | `bookTitle` |
| `summary` | `impression` (단일 필드로 통합) |
| `opinion` | (없음 — impression 에 포함) |
| `recommended` | (없음) |
| — | `pages` (BE only) |
| — | `appliedTo` (BE only — '어디에 적용했나') |

**영향**: prod 화면 진입 시 런타임 에러 (`undefined.title` 등). 학생이 독서 노트 화면 사용 불가.

**fix 규모**: 두 BookList.tsx 재작성 (~350 줄 × 2 = 700 줄 영향).

---

### 카테고리 B — 내가 만든 회귀: Homework shape 미스매치 (6 errors)

**파일**: `src/pages/middle-student/_pages/homework/Homework.tsx`

**원인**: PR #91 (homework swap) 작성 시 `SubmissionResponse` / `HomeworkResponse` 의 일부
필드 가정 오류.

| 코드 가정 | 실제 shape |
|-----------|-----------|
| `SubmissionResponse.score` | (다른 이름 — `gradedScore`? 확인 필요) |
| `HomeworkResponse.maxScore` | (없음 — 확인 필요) |

**영향**: 중등 숙제 화면 detail panel + status 계산이 broken.

**fix 규모**: Homework.tsx 일부 (~30 줄 수정).

---

### 카테고리 C — 기존 broken: master/questions (33 errors)

**파일**: `master/_pages/questions/*.tsx` (PastQuestionsTab 15 + MajorQuestionsTab 8 +
MajorDetailModal 7 + PassageInterviewTab 6 + PastDetailModal 5 + ExcelUploadModal 1 +
ExportModal 1)

**원인**:
1. `@/pages/master/_hooks/useMasterPastQuestions` 등 — **`_hooks/` 디렉토리 자체 부재** (이전
   `_hooks/` 컨벤션 deprecate 시 삭제됐는데 import 만 남음). 발생: task #196 시점.
2. `import * as XLSX from 'xlsx'` — **xlsx npm 패키지 미설치** (실제로 ExcelUploadModal /
   ExportModal 이 동작 안 함).
3. `TS7006 (implicit any)` 다수 — typed hook 으로 swap 안 됐기 때문.

**영향**: master/questions 화면이 mock-only 인데도 typecheck broken. master 진영 9 화면이
mock 인데, 그 중 questions 만 typecheck 깨짐.

**fix 규모**: 입시 도메인 BE swap PR 과 같이 처리 (master/PastQuestionsTab 완전 재작성 +
useMasterPastQuestions hook 신설). MajorQuestionsTab / PassageInterviewTab 은 BE 도메인
별도 신설 필요 (Major question / Passage interview).

---

### 카테고리 D — 기존 broken: concept atoms 부재 (7 errors)

**파일**: `high-student/_pages/concept/{CareerConcept,ConceptResult,ConceptWaiting}.tsx`
+ middle 동일 (3 + 3 + ConceptResult 2)

**원인**: `@/lib/auth/atoms` 모듈 부재. 이전 `studentAuth.ts` 로 통합되면서 path rename 누락.

**영향**: 진로 진단 결과 / 대기 화면이 broken (실제 화면 진입 시 모듈 로드 실패).

**fix 규모**: 7 파일 import path 수정 (`@/lib/auth/atoms` → `@/lib/auth/studentAuth`).
작은 작업 (10분).

---

### 카테고리 E — tsconfig verbatimModuleSyntax (2 errors)

**파일**: `src/lib/api/client.ts:11`

**원인**: `verbatimModuleSyntax: true` 설정 활성화 시점에 `import { AxiosInstance,
InternalAxiosRequestConfig }` 를 `import type` 로 분리 안 됨.

**fix**: `import axios, { type AxiosInstance, type InternalAxiosRequestConfig }` 1 줄 수정.

---

## 4. 우선순위 권장

| # | 카테고리 | 영향 | fix 비용 | 권장 |
|---|----------|------|----------|------|
| 1 | **A. BookList** | 학생 독서 노트 화면 prod 깨짐 | 中 (700 줄) | **즉시** |
| 2 | **B. Homework** | 중등 숙제 화면 prod 깨짐 | 小 (30 줄) | **즉시** |
| 3 | **D. concept atoms** | 진로 진단 일부 깨짐 | 小 (10분) | **즉시** |
| 4 | **E. verbatimModuleSyntax** | client.ts type import | 小 (1줄) | **즉시** |
| 5 | **C. master/questions** | mock 화면이라 prod 영향 없음 | 大 (BE 도메인 신설 동반) | 입시 도메인 PR 과 통합 |

## 5. CI 갱신 (PR #131 머지 완료)

**적용된 3종 fix** — CI 가 stale cache 로 silent 통과하던 root cause 봉쇄.

### 5.1. tsBuildInfoFile 경로 분리 (node_modules 외부)

`tsconfig.app.json` + `tsconfig.node.json`:
```diff
- "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo"
+ "tsBuildInfoFile": "./.cache/tsbuildinfo/tsconfig.app.tsbuildinfo"
```
→ setup-node 의 npm cache (`~/.npm`, `node_modules/`) 와 격리.

### 5.2. typecheck script 에 --force 추가

`package.json`:
```diff
- "typecheck": "tsc -b --noEmit"
+ "typecheck": "tsc -b --noEmit --force"
```
→ incremental `.tsbuildinfo` 무시. 매번 전수 검사 (약간 느려지지만 정확성 우선).

### 5.3. CI yml step 주석

`.github/workflows/ci.yml` Type check step 에 `--force` 의 의도 + 본 보고서 §6 참조 주석.

> 결과: clean CI run 에서 모든 src 검사 → 누적 회귀 silent 통과 불가능.

## 6. 발생 흐름 (postmortem)

1. PR #83 (BookList swap) 작성 시 local `npx tsc --noEmit` exit 0 → PASS 로 판단.
   CI 의 `tsc -b --noEmit` 는 .tsbuildinfo stale 캐시로 silent 통과.
2. PR #85 (middle BookList) 동일
3. PR #91 (Homework swap) 동일
4. 이후 모든 PR 이 동일 패턴 → 누적
5. 본 노트 작성 시점 (PR #119 이후) 누적 82 errors

**근본 원인 2단**:
1. **local 검증 명령 잘못** — `npx tsc --noEmit` (no -b, no -p) 는 root tsconfig 의
   `references` 를 따라가지 않음. 0 file 검사.
2. **CI incremental cache stale** — `tsc -b --noEmit` 가 정상이라도 `.tsbuildinfo` 가
   이전 PASS 상태로 캐싱되어 새 회귀를 검사 안 함. setup-node 의 npm cache 가
   `node_modules/.tmp/` 까지 같이 cache 하면서 PR 간 cache 가 유지됨.

두 요인이 동시에 작용 → "local PASS + CI PASS + 실제 broken" 의 비대칭 발생.

## 7. fix 진행 결과

| # | PR | 변경 핵심 |
|---|----|-----------|
| A | [#121](https://github.com/seum-running/bkurs-fe/pull/121) | BookList (high+middle) ReadingNoteView 실제 shape (bookTitle/author/pages/impression/appliedTo) + FeedbackView (content/recommendedScore) 로 재작성 |
| B | [#123](https://github.com/seum-running/bkurs-fe/pull/123) | Homework — submission.score → submission.grade, hw.maxScore 제거, hw.content → hw.description |
| D | [#125](https://github.com/seum-running/bkurs-fe/pull/125) | `src/lib/auth/atoms.ts` shim — `studentState`(=studentProfileAtom) + derived `academyState`. ConceptResult 미사용 prevConcept 제거 |
| E | [#127](https://github.com/seum-running/bkurs-fe/pull/127) | client.ts — AxiosInstance/InternalAxiosRequestConfig 에 `type` 키워드 |
| C | [#129](https://github.com/seum-running/bkurs-fe/pull/129) | master/questions 11 broken 파일 (PastQuestionsTab/MajorQuestionsTab/PassageInterviewTab/3 DetailModal/ExcelUploadModal/ExportModal/QuestionsStats/mock-data + lib/types/questions) 제거 + Questions.tsx 를 '준비 중' placeholder. router default import 정정 |
| CI | [#131](https://github.com/seum-running/bkurs-fe/pull/131) | tsBuildInfoFile → .cache/, package.json typecheck --force, ci.yml 주석 |

총 6 PR 머지. typecheck **82 → 0**. 입시 도메인 BE swap 시 master/questions 화면 (placeholder) 재작성 예정.

## 8. 재발 방지 체크리스트

신규 화면 / hook swap PR 작성 시:

- [ ] **local typecheck**: `npm run typecheck` (= `tsc -b --noEmit --force`) PASS
- [ ] **import 시 BE shape 직접 확인**: `src/lib/api/<domain>.ts` 의 interface 정의를 그대로 매핑 (가정 금지)
- [ ] **신규 export**: `index.ts` re-export 도 같이 — 사용처에서 `@/lib/api` 로 import 가능하게
- [ ] **CI green 확인 후 merge**: --force 가 새 회귀를 잡는지 확인

CI 변경 시:
- `.tsbuildinfo` 경로는 항상 `node_modules/` 외부로
- `tsc -b` 사용 시 CI 에서는 `--force` 권장 (local 은 incremental OK 지만 PR check 는 전수)

## 9. 관련 노트

- [[bkurs-fe]] — 프로젝트 hub
- [[screen-logic-and-api-map]] — 화면별 hook + endpoint 매핑
- [[../bkurs-be/getting-started]] — 진입점

## 10. 참고

- 권장 typecheck 명령: `cd ~/masterway-dev/bkurs-fe && npm run typecheck` (= `tsc -b --noEmit --force`)
- 단발 검사 대안: `npx tsc --noEmit -p tsconfig.app.json`
- 에러 카운트 재현: `npm run typecheck 2>&1 | grep error | wc -l`
- 파일별 카운트: `... | awk -F'(' '{print $1}' | sort | uniq -c | sort -rn`
- 에러 코드별: `... | grep -oE 'error TS[0-9]+' | sort | uniq -c | sort -rn`
