---
project: b-kurs-home
type: task-log
status: in-progress
created_at: 2026-06-02
updated: 2026-06-02
tags: [project, b-kurs-home, task-log, security, supabase, rls]
related:
  - "[[../b-kurs-home]]"
  - "[[../research/research-anon-key-exposure-issue-11]]"
  - "[[../decisions/decision-no-env-commit-use-example]]"
---

# 작업 로그 — anon key 노출/RLS 점검 + env 정책 정리 (Issue #11)

- 날짜: 2026-06-02
- 작업 레포: `seum-running/b-kurs-home`
- 원본 레포: `camshf-crypto/b-kurs` (PUBLIC)
- GitHub Issue: #11 (security: Supabase anon key 회전 및 RLS 점검)
- GitHub PR: (작성 후 갱신)
- Branch: `chore/env-policy-security-issue-11`
- 목표: 노출 여부 확인 + 회전/ RLS 점검 판단을 vault 에 기록하고, 코드 레포에는 env 보안 정책 문서만 정리.

## 참고한 에이전트 규칙 문서

- `policies/reference/COMMIT_CONVENTION.md`, `BRANCH_STRATEGY.md`
- 전역 안전 규칙: secret/자격 정보 커밋 금지
- 자율 머지 정책(클론/이관 기간): 조건 충족 시 merge commit 자율 머지

## 변경 요약

(작업 진행 중 — 커밋 후 갱신)

## 주요 결정 / 조사 결과

- 노출: 원본 PUBLIC 레포 `.env` 에 anon/publishable key + URL (2개)뿐.
  `service_role` 등 고위험 비밀 노출 없음. `b-kurs-home` 에는 키가 history/tree
  모두에 없음(확인). 상세 → [[../research/research-anon-key-exposure-issue-11]].
- 회전: 우선순위 낮음(공개 전제 키). RLS 점검이 우선. 회전은 RLS 오설정 시 또는
  운영 위생 차원에서 operator 가 결정.
- RLS: `contact_inquiries` = anon INSERT-only + RLS 활성화가 목표 상태.
  **라이브 검증은 대시보드 접근 필요로 보류 (검증 못함)** — operator 작업.

## 버린 대안

- 코드 PR 에서 실제 회전/RLS 토글 시도 — 대시보드 접근/실 자격 필요해 범위 밖. 보류.
- anon key 즉시 강제 회전 — 공개 전제 키라 이득 제한적, RLS 점검 선행이 합리적.

## 검증 결과

- local: lint 0 / typecheck 0 / build 성공 (문서/config 변경, 동작 무영향).
- CI: (PR 후 갱신)
- secret 미기록 확인: 본 작업의 어떤 문서/커밋에도 실 키 값 없음.

## 남은 리스크 / 보류

- 라이브 RLS 미검증 (operator 대시보드 접근 필요).
- anon key 회전 여부 미결정 (operator 판단).

## 다음 작업

- operator: 대시보드 RLS 실측 → Issue #11 코멘트 + research 노트 §5 갱신.
- 필요 시 anon key 회전 + Vercel/로컬 env 갱신.
