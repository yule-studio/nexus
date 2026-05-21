---
title: bkurs-fe — 누적 typecheck 회귀 분석 (82 errors, 2026-05-21)
kind: report
status: open
created_at: 2026-05-21
tags: [bkurs-fe, regression, typecheck, postmortem]
related:
  - "[[bkurs-fe]]"
  - "[[screen-logic-and-api-map]]"
home_hub: "[[bkurs-fe]]"
---

# bkurs-fe 누적 typecheck 회귀 보고서 (2026-05-21)

> ⚠️ **본 노트는 진단 보고서다.** 수정 PR 은 별도. 사용자 우선순위 결정 후 fix 진행 예정.

## 1. 한 줄 요약

`tsc --noEmit -p tsconfig.app.json` 실행 시 **82 errors** 가 `src/` 안에 존재. 이전 PR 들이
`npx tsc --noEmit` (root tsconfig 만 검사 — `references` 만 있고 `files: []` 라 사실상
0 file 검사) 으로 검증해 통과로 보였으나, 실제 빌드 / IDE 는 에러를 본다.

## 2. 검증 명령 차이

| 명령 | 검사 범위 | 결과 |
|------|-----------|------|
| `npx tsc --noEmit` | root tsconfig `files: []` → **0 file** | exit 0 (잘못된 PASS) |
| `npx tsc --noEmit -p tsconfig.app.json` | `src/` 전체 | exit 0 + **82 errors 출력** |
| `npx tsc -b` | composite (app + node) | (확인 필요) |

> tsc 가 exit 0 을 반환해도 stdout 에 에러를 뱉으면 실제로는 fail. CI 가 stdout 검사 안 하면
> 통과로 잘못 보고됨. **즉시 CI 갱신 필요**.

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

## 5. 즉시 조치 — CI 갱신

bkurs-fe CI `.github/workflows/*.yml` 의 typecheck 단계:

```diff
- npx tsc --noEmit
+ npx tsc --noEmit -p tsconfig.app.json
```

또는 `package.json`:
```json
{
  "scripts": {
    "typecheck": "tsc --noEmit -p tsconfig.app.json"
  }
}
```

> CI 가 통과로 잘못 보고하던 것을 막아야 본 회귀가 또 발생 안 함.

## 6. 발생 흐름 (postmortem)

1. PR #83 (BookList swap) 작성 시 `npx tsc --noEmit` exit 0 보고 PASS 로 판단
2. PR #85 (middle BookList) 동일
3. PR #91 (Homework swap) 동일
4. 이후 모든 PR 이 동일 명령으로 검증 → 누적
5. 본 노트 작성 시점 (PR #119 이후) 누적 82 errors

**근본 원인**: tsc 명령이 root tsconfig 의 `files: []` 를 그대로 받아 0 file 검사. 이전
프로젝트들과 패턴 다름 (보통 root tsconfig 가 `include: ["src"]`).

## 7. 다음 액션 (fix PR 시리즈)

1. **fix-A**: BookList (high + middle) ReadingNoteView 실제 shape 로 재작성. `useMyNoteFeedback`
   응답 (FeedbackView) 도 검토 — `content`/`recommendedScore` 정확한지.
2. **fix-B**: Homework Submission/Homework 실제 shape 확인 후 수정.
3. **fix-D**: concept atoms import path 수정 (4 파일).
4. **fix-E**: client.ts type import 분리.
5. **CI**: typecheck script 갱신 (사용자 결정 후).
6. **fix-C** (선택): 입시 도메인 BE 신설 + master/PastQuestionsTab 재작성 통합 PR 로.

## 8. 관련 노트

- [[bkurs-fe]] — 프로젝트 hub
- [[screen-logic-and-api-map]] — 화면별 hook + endpoint 매핑 (DONE/MOCK/N/A status 재검토 필요)
- [[../bkurs-be/getting-started]] — 진입점

## 9. 참고

- typecheck 명령: `cd ~/masterway-dev/bkurs-fe && npx tsc --noEmit -p tsconfig.app.json`
- 에러 카운트 재현: `npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep error | wc -l`
- 파일별 카운트: `... | awk -F'(' '{print $1}' | sort | uniq -c | sort -rn`
- 에러 코드별: `... | grep -oE 'error TS[0-9]+' | sort | uniq -c | sort -rn`
