---
title: "bkurs-fe pages — middle-student (중등 학생)"
kind: handover
status: stable
created_at: 2026-05-22T01:28:00+09:00
tags: [bkurs-fe, handover, pages, middle-student]
related:
  - "[[catalog]]"
  - "[[_shared]]"
home_hub: "[[../../bkurs-fe]]"
---

# middle-student (중등 학생) 페이지

> BE Role: `STUDENT_MIDDLE`. brand 색: brand-middle `#10B981` (초록). 부트스트랩 4 + 학습 15 = 19 화면.

## 1. 디렉토리

```
pages/middle-student/
├── _layout/StudentLayout.tsx
├── _components/ThreadView.tsx          # middle 전용 (4 페이지 재사용)
├── _data/                              # mock
│   ├── student-mock.ts / expect-mock.ts / extras-mock.ts
└── _pages/
    ├── login/Login.tsx                 # useStudentLogin('middle')
    ├── signup/Signup.tsx
    ├── connect/Connect.tsx
    ├── pending/Pending.tsx
    ├── roadmap/Roadmap.tsx
    ├── lesson/Lesson.tsx                # course
    ├── concept/CareerConcept.tsx
    ├── book/BookList.tsx                # reading
    ├── homework/Homework.tsx            # homework — 중등 전용 UI
    ├── expect/
    │   ├── Expect.tsx                   # AiAnalysisPanel + ThreadView
    │   └── EssayWizard.tsx               # expect 의 sub-route
    ├── past/Past.tsx                    # AiAnalysisPanel + ScoreRadar
    ├── presentation/Presentation.tsx    # AiAnalysisPanel + ScoreRadar + ThreadView
    ├── simulation/Simulation.tsx
    ├── debate/Debate.tsx                # discussion
    ├── suhaeng/Suhaeng.tsx              # AiAnalysisPanel + ThreadView
    └── record/Record.tsx                # saenggibu (중등 생기부)
```

## 2. 페이지 ↔ hook ↔ BE endpoint

### 부트스트랩 4 — [[_shared]] 와 동일 (`useStudent*('middle')`)

### 학습 페이지 (15)

| path | 컴포넌트 | hook | BE 매핑 |
| --- | --- | --- | --- |
| `/middle-student/roadmap` | `MiddleStudentRoadmap` | (FE) | — |
| `/middle-student/lesson` | `MiddleStudentLesson` | (후속) | course |
| `/middle-student/concept` | `MiddleStudentConcept` | useConceptDiagnosis | `/v1/students/me/concept-diagnosis` |
| `/middle-student/book` | `MiddleStudentBookList` | useReading | `/v1/students/me/reading-notes` |
| `/middle-student/homework` | `MiddleStudentHomework` | useHomework + useSubmission | `/v1/students/me/homework`, `/submit` |
| `/middle-student/expect` | `MiddleStudentExpect` | useQuestion (EXPECTED) | `/v1/students/me/questions?type=EXPECTED` |
| `/middle-student/expect/wizard` | `MiddleStudentEssayWizard` | (sub-route) | |
| `/middle-student/past` | `MiddleStudentPast` | usePastQuestion | `/v1/students/me/past-questions` |
| `/middle-student/presentation` | `MiddleStudentPresentation` | (mock + interview 후속) | — |
| `/middle-student/simulation` | `MiddleStudentSimulation` | useSimulation | `/v1/students/me/simulation-cases` |
| `/middle-student/debate` | `MiddleStudentDebate` | useDiscussion | `/v1/students/me/discussions` |
| `/middle-student/suhaeng` | `MiddleStudentSuhaeng` | useSuhaeng | `/v1/students/me/suhaeng` |
| `/middle-student/record` | `MiddleStudentRecord` | useSaenggibu | `/v1/students/me/saenggibu` |

## 3. 인증 / 가드 (학생 4 status)

high-student 와 동일 흐름 — `_shared/auth/useStudent*('middle')`. 자세히 → [[high-student]] §3.

## 4. 공유 컴포넌트

- **AiAnalysisPanel** — expect / past / presentation / suhaeng 4 페이지
- **ScoreRadar** — past / presentation 2 페이지
- **ThreadView (middle 전용)** — expect / past / presentation / suhaeng 4 페이지

자세히 → [[../modules/shared-components]]

## 5. high-student 와의 차이

| 항목 | high | middle |
| --- | --- | --- |
| 색 | brand-high (파랑) | brand-middle (초록) |
| 생기부 | record (8 카테고리 + bridge) | saenggibu (학기 free-form + lock) |
| 페이지 수 | 13 | 15 (homework / debate / EssayWizard 등 추가) |
| ThreadView 사용 | 3 페이지 | 4 페이지 |

## 6. cross-link

- [[catalog]]
- [[_shared]]
- [[high-student]]
- [[../api-clients/saenggibu]] — 중등 생기부 BE
- [[../api-clients/homework]] / [[../api-clients/discussion]] / [[../api-clients/reading]] / [[../api-clients/question]] / [[../api-clients/pastQuestion]] / [[../api-clients/concept]] / [[../api-clients/simulation]] / [[../api-clients/suhaeng]]
