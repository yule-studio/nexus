---
title: bkurs-fe — Supabase → bkurs-be API 전환 기반 (S1 ~ S3)
kind: architecture
status: active
created_at: 2026-05-21
tags: [bkurs-fe, architecture, api, axios, envelope]
related:
  - "[[bkurs-fe]]"
  - "[[architecture-async-mq-batch]]"
  - "[[spring-response-policy]]"
home_hub: "[[bkurs-fe]]"
---

# bkurs-fe — Supabase → bkurs-be API 전환 기반

## 1. 핵심 요약

bkurs-fe 는 inroad-high 원본 시점에 supabase 클라이언트를 직접 호출했다.
bkurs-be 가 도달하면서 **점진 전환** 을 위해 다음 3 단계를 정의한다.

| 단계 | 범위 | 상태 |
|------|------|------|
| **S1** | `src/lib/api/` 기반 (envelope / apiClient / 도메인 client) | **DONE (#60)** |
| **S2** | conceptApi 위에 react-query hook 격리 | **DONE (#64)** — 화면 swap 은 S3 |
| **S3** | 화면이 useConceptDiagnosis hook 호출 → supabase 제거 | 후속 issue (BE dev 환경 도달 후) |

S1 은 **구조** 만 깔고 화면 동작은 supabase 그대로 유지 — 회귀 위험 최소화.

## 2. 내 해석 (왜 이 구조)

### 2.1. axios + Bearer interceptor 가 표준인 이유

bkurs-be 는 Spring Security 6 + JWT (HS256) 로 모든 보호 endpoint 가
`Authorization: Bearer <token>` 을 요구한다. 매 호출마다 헤더를 손으로
조립하면 (1) 누락 위험 (2) refresh 흐름 추가 시 모든 호출 지점을 다시
패치해야 함. 따라서 **interceptor 한 곳** 에 부착 책임 통합.

대안:
- **fetch + 각 화면 wrapper** — 표준 라이브러리만 쓰는 장점이 있지만
  interceptor 패턴이 없어 401 fallback 을 각 화면이 따로 구현.
- **react-query 만** — 캐싱은 좋지만 raw transport 책임은 여전히 axios
  /fetch 가 필요. 이건 S2 cycle 에서 추가.

### 2.2. ApiEnvelope unwrap 책임을 도메인 client 에 둔 이유

bkurs-be 의 모든 200 응답은 `{success, data, meta}` 형태.
화면이 `res.data.data` 처럼 envelope 안을 직접 뚫고 들어가면 (1) 타입
안정성 깨짐 (2) error 응답 처리 누락. 도메인 client (`conceptApi.getMine()`)
가 `unwrap()` 책임을 가져가면 화면은 `T` 만 보면 된다.

`unwrap()` 은 envelope.success === false 면 `ApiClientError` throw —
화면은 try/catch 만으로 4xx/5xx 를 일관 처리.

### 2.3. 학생 / 강사 / 마스터 토큰을 같은 storage key 로 묶은 이유

`bkurs.accessToken` 한 키로 묶었다. 이유: 한 브라우저에서 두 role 로
동시 로그인하는 케이스는 없으며 (실제로는 로그아웃 후 재로그인),
storage key 를 분리하면 interceptor 가 어느 키를 읽을지 분기해야 함.
S3 에서 role 별 분리가 필요해지면 그 시점에 storage namespace 추가.

### 2.4. 화면 자체를 즉시 교체하지 않은 이유 (가장 중요)

inroad-high 의 supabase 스키마와 bkurs-be 의 B30 스키마는 데이터 모델이
다르다:

| 영역 | supabase (`student_concept`) | bkurs-be (`concept_diagnosis`) |
|------|------------------------------|--------------------------------|
| primary key | `(student_id, academy_id, grade)` multi-row | `student_account_id` uk (학생 1인 1행) |
| 학년 분리 | row 분리 | 단일 row (학년은 student 프로필) |
| 학원 분리 | row 분리 | 단일 row (학원은 enrollment) |
| answers | JSONB `answers` | TEXT `answers_json` |

화면을 한 번에 교체하면 (1) 학년별 데이터 분리가 깨지거나 (2) BE 스키마
재설계 필요. S3 cycle 에서 사용자와 **학년별 모델 vs 단일 모델** 정책
결정 후 진행. 그 전까지는 구조만 깔아두고 화면은 supabase 호환.

## 3. 적용 맥락

### 3.1. 새 도메인 client 추가 절차 (template)

```ts
// src/lib/api/<domain>.ts
import { apiClient } from './client'
import { unwrap, type ApiEnvelope } from './envelope'

export interface FooView { /* BE response DTO mirror */ }

export const fooApi = {
  async getMine(): Promise<FooView> {
    const res = await apiClient.get<ApiEnvelope<FooView>>('/v1/students/me/foo')
    return unwrap(res.data)
  },
}
```

`src/lib/api/index.ts` 에 export 추가 → 화면에서 `import { fooApi } from '@/lib/api'`.

### 3.2. 인증 토큰 set / clear 시점

- 로그인 성공 시: `setAccessToken(response.accessToken)`
- 로그아웃 / 401 응답 시: `setAccessToken(null)` (401 은 interceptor 가 자동)
- refresh 흐름은 S2 cycle 에서 `apiClient.interceptors.response` 에 추가.

### 3.3. env 변수

`VITE_API_BASE_URL` — bkurs-be base URL. dev 환경 도달 전까지 기본값 `/api`
(vite proxy 로 로컬 8080 forward 가능).

## 3.4. S2 — react-query hook 패턴 (#64)

```ts
// src/lib/hooks/useConceptDiagnosis.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { conceptApi, ... } from '@/lib/api'

const CONCEPT_KEY = ['concept', 'mine'] as const

export function useConceptDiagnosis() {
  return useQuery({ queryKey: CONCEPT_KEY, queryFn: () => conceptApi.getMine() })
}

export function useUpdateConceptProgress() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (req) => conceptApi.updateProgress(req),
    onSuccess: (data) => qc.setQueryData(CONCEPT_KEY, data),
  })
}
```

원칙:
- 쿼리 키는 도메인 + 의미 (`['concept', 'mine']`, `['concept', 'teacher', sid]`).
- mutation 의 onSuccess 는 같은 자원의 쿼리 캐시를 `setQueryData` 로 갱신
  (`invalidateQueries` 보다 한 라운드 트립 절약).
- 강사용 쿼리는 `enabled: sid > 0` 으로 boot 시 placeholder fetch 방지.

## 4. 관련 노트

- [[architecture-async-mq-batch]] — bkurs-be 의 비동기 / MQ / 배치 구조
- [[spring-response-policy]] — ApiEnvelope 표준 자체
- [[bkurs-fe]] — bkurs-fe 프로젝트 메인 허브

## 5. 참고

- 코드: `bkurs-fe/src/lib/api/{envelope,client,concept,index}.ts`
- BE 매칭: `bkurs-be/src/main/java/com/bkurs/concept/controller/*Controller.java`
- 후속: 화면별 hook 분리 (S2) → conceptApi 실 호출 (S3).
