---
title: "bkurs-fe — 22 API client 카탈로그"
kind: handover
status: stable
created_at: 2026-05-22T01:05:00+09:00
tags: [bkurs-fe, handover, api-client, react-query]
related:
  - "[[../../bkurs-fe]]"
  - "[[../modules/lib-layer-guide]]"
  - "[[../../bkurs-be/handover/domains/catalog]]"
home_hub: "[[../../bkurs-fe]]"
---

# bkurs-fe — 22 API client 카탈로그

> `lib/api/{domain}.ts` 25 파일 (22 도메인 + client.ts + envelope.ts + index.ts). 본 인덱스는 22
> 도메인 client 의 책임 + BE 매핑 + 사용처 + sub doc 링크. 각 client 의 메서드 시그니처 / hook /
> 사용 페이지는 `api-clients/{name}.md`. 컨벤션은 [[../modules/lib-layer-guide]].

## 0. 한 장 인덱스

| client | 카테고리 | BE 도메인 | 책임 1줄 | hook | sub doc |
| --- | --- | --- | --- | --- | --- |
| auth | core | [[../../bkurs-be/handover/domains/auth\|auth]] | 로그인 / signup / refresh / me / changePassword | useAuth | [[auth]] |
| academy | core | [[../../bkurs-be/handover/domains/academy\|academy]] | 학원 + membership + 강사 invite + 이관 | useAcademy | [[academy]] |
| announcement | core | [[../../bkurs-be/handover/domains/announcement\|announcement]] | 학원 공지 CRUD + read mark | useAnnouncement | [[announcement]] |
| studentProfile | core | [[../../bkurs-be/handover/domains/student\|student]] | 학생 프로필 upsert/patch/get | useStudentProfile | [[studentProfile]] |
| plan | core | [[../../bkurs-be/handover/domains/plan\|plan]] | 플랜 / 구독 / 결제 / 환불 | usePlan | [[plan]] |
| homework | learning | [[../../bkurs-be/handover/domains/homework\|homework]] | 숙제 + 제출 + 채점 | useHomework | [[homework]] |
| statement | learning | [[../../bkurs-be/handover/domains/statement\|statement]] | 자소서 + feedback | useStatement | [[statement]] |
| question | learning | [[../../bkurs-be/handover/domains/question\|question]] | 강사 발행 예상/기출 + attempt | useQuestion | [[question]] |
| interview | learning | [[../../bkurs-be/handover/domains/interview\|interview]] | 모의면접 + 미디어 응답 + 평가 | useInterview | [[interview]] |
| record | learning | [[../../bkurs-be/handover/domains/record\|record]] | 학생부 8 카테고리 (read + manual write) | useRecord | [[record]] |
| discussion | learning | [[../../bkurs-be/handover/domains/discussion\|discussion]] | 토론 topic + post upsert + 평가 | useDiscussion | [[discussion]] |
| reading | learning | [[../../bkurs-be/handover/domains/reading\|reading]] | 독서 노트 + feedback | useReading | [[reading]] |
| performance | learning | [[../../bkurs-be/handover/domains/performance\|performance]] | 수행평가 + 제출 + 채점 | usePerformance | [[performance]] |
| simulation | learning | [[../../bkurs-be/handover/domains/simulation\|simulation]] | 모의지원 + response + review | useSimulation | [[simulation]] |
| search | aggregator | [[../../bkurs-be/handover/domains/search\|search]] | 5 도메인 통합 검색 | useSearch (후속) | [[search]] |
| stats | aggregator | [[../../bkurs-be/handover/domains/stats\|stats]] | 학생 7 영역 통계 | useStats (후속) | [[stats]] |
| dashboard | aggregator | [[../../bkurs-be/handover/domains/dashboard\|dashboard]] | 강사 / 학원장 대시보드 | useDashboard | [[dashboard]] |
| concept | content | [[../../bkurs-be/handover/domains/concept\|concept]] | 진로 진단 (중·고등 공용) | useConceptDiagnosis | [[concept]] |
| suhaeng | content | [[../../bkurs-be/handover/domains/suhaeng\|suhaeng]] | 중등 수행/면담 round | useSuhaeng | [[suhaeng]] |
| saenggibu | content | [[../../bkurs-be/handover/domains/saenggibu\|saenggibu]] | 중등 생기부 + lock | useSaenggibu | [[saenggibu]] |
| pastQuestion | content | [[../../bkurs-be/handover/domains/admission\|admission]] | 입시 기출 (MASTER 발행) | usePastQuestion | [[pastQuestion]] |
| majorQuestion | content | [[../../bkurs-be/handover/domains/admission\|admission]] | 입시 학과 면접 (MASTER 발행) | useMajorQuestion | [[majorQuestion]] |

> file/pdf/notification/ai/stt 의 BE gateway 5종은 FE 직접 호출 없음 — BE 가 처리.

## 1. 카테고리별

### core (5)
계정 / 학원 / 학생 — 모든 화면이 의존.

### learning (9)
강사 + 학생 학습 활동. record / discussion / reading / performance / simulation 의 grade/evaluate
이벤트는 BE 측에서 record bridge 로 자동 학생부 반영.

### aggregator (3)
read-only 집계 client. 학생 1인 stats / 학원 dashboard / 5 도메인 search.

### content (5)
공통 / 중등 전용 / MASTER 발행 (입시 문항).

## 2. 인프라 파일 (3)

| 파일 | 역할 |
| --- | --- |
| `lib/api/client.ts` | axios 인스턴스 + Bearer 자동 부착 + 401 처리 |
| `lib/api/envelope.ts` | ApiEnvelope / ApiError / ApiClientError + unwrap() |
| `lib/api/index.ts` | re-export |

자세히 → [[../modules/lib-layer-guide]] §1~2.

## 3. 컨벤션 (요약)

```ts
// lib/api/{domain}.ts
export const {domain}Api = {
  async listMine(): Promise<...> { ... },
  async create(req): Promise<...> { ... },
  // ...
}

// lib/hooks/use{Domain}.ts
const KEY = ['domain', 'mine'] as const
export function useMy{Domain}() {
  return useQuery({ queryKey: KEY, queryFn: () => api.listMine() })
}
```

자세한 query key 규칙은 [[../decisions/decision-hook-convention]].

## cross-link

- [[../../bkurs-fe]] — 프로젝트 hub
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
- [[../../bkurs-be/handover/domains/catalog]] — BE 29 도메인 (gateway 5 + batch + common 제외 시 22 = FE 매핑)
- [[../decisions/decision-hook-convention]] — hook 위치 / query key ADR
