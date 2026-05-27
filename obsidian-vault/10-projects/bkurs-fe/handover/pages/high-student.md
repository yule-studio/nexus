---
title: "bkurs-fe pages — high-student (고등 학생)"
kind: handover
status: stable
created_at: 2026-05-22T01:27:00+09:00
tags: [bkurs-fe, handover, pages, high-student]
related:
  - "[[catalog]]"
  - "[[_shared]]"
home_hub: "[[../../bkurs-fe]]"
---

# high-student (고등 학생) 페이지

> BE Role: `STUDENT_HIGH`. brand 색: brand-high `#2563EB` (파랑). 부트스트랩 4 + 학습 13 = 17 화면.

## 1. 디렉토리

```
pages/high-student/
├── _layout/StudentLayout.tsx          # 인증 + status 가드 + 사이드바
├── _components/ThreadView.tsx          # high 전용 (3 페이지 재사용)
├── _data/                              # mock (점진 제거)
│   ├── student-mock.ts / student-mock2.ts / media-mock.ts
└── _pages/
    ├── login/Login.tsx                 # useStudentLogin('high')
    ├── signup/Signup.tsx                # useStudentSignup('high')
    ├── connect/Connect.tsx              # useStudentConnect('high')
    ├── pending/Pending.tsx              # useStudentPending('high')
    ├── roadmap/Roadmap.tsx              # FE only
    ├── topic/TopicList.tsx              # discussion
    ├── book/BookList.tsx                # reading
    ├── record/Record.tsx                # 생기부 본인 조회
    ├── major/Major.tsx                  # admission/major
    ├── expect/Expect.tsx                # question (EXPECTED) — AiAnalysisPanel + ThreadView
    ├── past/Past.tsx                    # admission/past — AiAnalysisPanel + ThreadView
    ├── concept/CareerConcept.tsx        # concept
    ├── presentation/Presentation.tsx    # (FE) — ThreadView
    ├── mockexam/Mockexam.tsx            # interview
    ├── simulation/Simulation.tsx        # simulation
    └── suhaeng/Suhaeng.tsx              # suhaeng (중등 위주이지만 고등도 사용)
```

## 2. 페이지 ↔ hook ↔ BE endpoint

### 부트스트랩 (4 + _shared/auth/ 4 hook)

| path | 컴포넌트 | hook (_shared) | BE endpoint |
| --- | --- | --- | --- |
| `/high-student/login` | `HighStudentLogin` | useStudentLogin('high') | `POST /api/v1/auth/login` |
| `/high-student/signup` | `HighStudentSignup` | useStudentSignup('high') | `POST /api/v1/auth/signup` |
| `/high-student/connect` | `HighStudentConnect` | useStudentConnect('high') | `POST /api/v1/auth/connect` |
| `/high-student/pending` | `HighStudentPending` | useStudentPending('high') | `GET /api/v1/auth/me` (polling) |

자세히 → [[_shared]] (4 hook 공통)

### 학습 페이지 (13)

| path | 컴포넌트 | hook | BE 매핑 |
| --- | --- | --- | --- |
| `/high-student/roadmap` | `Roadmap` | (FE) | — |
| `/high-student/topic` | `TopicList` | useDiscussion ([[../api-clients/discussion]]) | `/v1/students/me/discussions` |
| `/high-student/book` | `BookList` | useReading | `/v1/students/me/reading-notes` |
| `/high-student/record` | `Record` | useRecord | `/v1/students/me/records` |
| `/high-student/major` | `Major` | useMajorQuestion | `/v1/students/me/major-questions` |
| `/high-student/expect` | `Expect` | useQuestion (EXPECTED) | `/v1/students/me/questions?type=EXPECTED` |
| `/high-student/past` | `Past` | usePastQuestion | `/v1/students/me/past-questions` |
| `/high-student/concept` | `CareerConcept` | useConceptDiagnosis | `/v1/students/me/concept-diagnosis` |
| `/high-student/presentation` | `Presentation` | (FE / interview 후속) | — |
| `/high-student/mockexam` | `Mockexam` | useInterview (후속) | `/v1/students/me/interviews` |
| `/high-student/simulation` | `Simulation` | useSimulation | `/v1/students/me/simulation-cases` |
| `/high-student/suhaeng` | `Suhaeng` | useSuhaeng | `/v1/students/me/suhaeng` |

## 3. 인증 / 가드 (학생 4 status)

`StudentLayout` 가드 흐름 ([[../architecture/routing-and-protection]] §3.3):

```
status = PENDING_CONNECT → /<high-student>/connect
status = PENDING_APPROVE → /<high-student>/pending
status = ACTIVE          → /<high-student>/roadmap (또는 의도된 path)
status = REJECTED        → login + 에러 메시지
```

`studentTokenAtom` + `studentProfileAtom` (lib/auth/studentAuth — high/middle 공용).

## 4. 공유 컴포넌트

- **AiAnalysisPanel** — expect / past 페이지 (admin/_pages 측 import)
- **ThreadView (high 전용)** — expect / past / presentation 3 페이지
- 자세히 → [[../modules/shared-components]]

## 5. cross-link

- [[catalog]]
- [[_shared]] — 4 부트스트랩 hook
- [[middle-student]] — 중등 대응 (디자인은 brand-middle 초록)
- [[../api-clients/auth]] / [[../api-clients/discussion]] / [[../api-clients/reading]] / [[../api-clients/record]] / [[../api-clients/question]] / [[../api-clients/pastQuestion]] / [[../api-clients/majorQuestion]] / [[../api-clients/concept]] / [[../api-clients/interview]] / [[../api-clients/simulation]] / [[../api-clients/suhaeng]]
