---
title: "bkurs-fe — 데이터 계층 선택 (mock / Supabase / 별도 백엔드 API)"
kind: research
project: bkurs-fe
session_id: bkurs-fe-setup
created_at: 2026-05-20T00:25:00+09:00
agent: engineering-agent/tech-lead
status: complete
related:
  - "[[../bkurs-fe]]"
  - "[[../decisions/decision-vite-react-19-tailwind-antd-jotai]]"
  - "[[../../bkurs-be/bkurs-be]]"
tags: [research, frontend, data-layer, backend-api]
---

# 질문

inroad-high (원본) 는 컴포넌트가 Supabase client 를 직접 호출 (`supabase.from(...)`). bkurs-fe 의 데이터 계층을 어떻게 가져갈 것인가?

# 비교

| 옵션 | 장점 | 단점 | 적합도 |
|---|---|---|---|
| **A. Supabase 직접 호출 (원본 유지)** | 가장 빠른 개발, BaaS 의 auth/storage/edge fn 모두 활용 | RLS 만으로 학원 도메인 정책 표현 한계. 외부 SDK (Claude / STT / 카카오) 호출이 클라에서 노출. 결제·회계 데이터 신뢰 경계 불명확 | ⛔ |
| **B. 별도 백엔드 API (REST/JSON)** | 도메인 정책 / 외부 SDK / 회계 모두 신뢰 경계 명확. mock → axios hook 으로 한 줄씩 교체 | 백엔드 개발 / 운영 비용 추가. 초기 협의 비용 | ✅ |
| **C. 하이브리드 (auth = Supabase, 도메인 = API)** | auth 빠른 도입 | 두 인증 시스템 동기화 비용 | ⛔ (운영 복잡) |

# 결정

**B 선택** + 단계적 도입.

## 단계

1. **mock 단계 (지금)**: 모든 데이터는 `*_MOCK` 상수, mutation 은 alert. 화면 우선 완성.
   - 28 PR (S2A~S2N) 완료 — vault [[../task-logs/task-log-s2-series-summary]]
2. **백엔드 도입 단계 (다음)**: [[../../bkurs-be/bkurs-be]] 의 endpoint 가 준비되면 hook 한 줄씩 교체:
   - `STUDENT_ESSAYS_MOCK.find(...)` → `useStudentEssays(studentId)` → `axios.get('/api/students/:id/essays')`
   - 한 도메인 = 한 PR
3. **인증 도입 단계**: 백엔드와 협의 (JWT / 세션 미정)

# Why (4 구조)

**왜 필요한가**: 인프라 선택은 처음에 결정 안 하면 docstring / 데이터 흐름 / 보안 경계가 일관 안 됨.

**안 하면 무엇이 깨지는가**: Supabase 유지하면 — Claude API key 가 클라에서 노출 / 결제 로직이 RLS 만으로 보호 불가 / 학원 정책 (학생 승인 흐름) 코드가 RLS SQL 에 흩어짐.

**대안 비교**: 위 표.

**트레이드오프**: B 는 초기 협의 비용 + 백엔드 운영 비용 추가. 다만 학원 SaaS 의 신뢰 경계 / 외부 SDK 통제 / 결제 회계 모두 B 가 필수.

## 회고

- mock 단계 docstring 에 "S3 = Supabase 단계" 표현이 누적됨. 사용자가 2026-05-19 정정 ("Supabase 아니고 백엔드 API"). 일괄 정정 작업이 추가 필요 (TODO)
- mock 데이터 타입 (StudentEssay, PastQuestion 등) 이 API 응답 스키마 초안으로 활용 → 백엔드와 협의 시 시작점이 됨

# 참고

- [[../../bkurs-be/bkurs-be|짝 백엔드 프로젝트]]
- [[../decisions/decision-vite-react-19-tailwind-antd-jotai]]
