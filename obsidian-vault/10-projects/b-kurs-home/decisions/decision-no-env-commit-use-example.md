---
project: b-kurs-home
type: decision
status: accepted
created_at: 2026-06-02
updated: 2026-06-02
tags: [project, b-kurs-home, adr, security, env]
related:
  - "[[../research/initial-repo-audit]]"
  - "[[../b-kurs-home]]"
---

# ADR: 실 `.env` 비커밋, `.env.example` 로 대체

## 상태

Accepted (2026-06-02)

## 맥락

원본 `camshf-crypto/b-kurs` (public 레포) 는 실 `.env` 를 커밋했다:

- `VITE_BKURS_SUPABASE_URL`
- `VITE_BKURS_SUPABASE_ANON_KEY` (publishable anon key)

`VITE_` prefix 변수는 빌드 시 클라이언트 번들에 인라인되므로 기술적으로
브라우저에 노출되는 값이다. 즉 supabase anon/publishable key 는 본래
공개를 전제로 한 키다. 그럼에도:

- 실 값을 VCS 에 커밋하면 (a) 키 회전이 git history 에 남고, (b) 어떤 변수가
  필요한지/예시값이 무엇인지 문서화가 안 되고, (c) 실수로 service_role 같은
  진짜 비밀이 같은 패턴으로 커밋되는 사고를 유발한다.
- 에이전트 전역 안전 규칙: "secret / 자격 정보 / 로컬 runtime state 절대 커밋 금지."

## 결정

1. `b-kurs-home` 으로 **실 `.env` 를 이관하지 않는다.**
2. placeholder 값만 담은 `.env.example` 을 추가한다.
3. `.gitignore` 에 `.env`, `.env.*` 를 명시 제외하고 `.env.example` 만 예외(`!.env.example`)로 추적한다.
4. 변수 의미/획득 방법은 `README.md` + `.env.example` 주석에 문서화한다.

## 대안과 트레이드오프

- **(대안 A) 원본처럼 실 `.env` 커밋** — 즉시 실행은 편하나 위 보안/문서화
  문제 그대로. 기각.
- **(대안 B) GitHub Secrets/Vercel env 만 쓰고 example 도 안 둠** — 로컬
  온보딩이 어려워짐 (어떤 변수가 필요한지 모름). 기각.
- **(채택) `.env.example` + gitignore** — 온보딩 문서화 + 실 값 비노출.
  트레이드오프: clone 후 `cp .env.example .env.local` 한 단계 필요 (README 에 명시).

## 후속 (본 PR 범위 밖)

- 원본 public 레포에 노출된 anon key 는 Supabase 콘솔에서 회전 권장.
  publishable key 라 위험도는 낮으나, RLS 정책 점검과 함께 후속 작업으로 둔다.
- **추적: Issue #11** (security) 로 승격. 조사 결과(노출 범위/회전 판단/RLS 목표
  상태) → [[../research/research-anon-key-exposure-issue-11]],
  작업 로그 → [[../task-logs/task-log-security-anon-key-rls-issue-11]].
  결론: anon=공개 전제 키라 회전 우선순위 낮음, **RLS 점검 우선** (라이브 검증은
  대시보드 접근 필요로 보류). service_role 등 고위험 비밀 노출은 없음(확인).
