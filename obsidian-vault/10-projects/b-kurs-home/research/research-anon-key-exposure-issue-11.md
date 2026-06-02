---
project: b-kurs-home
type: research
status: done
created_at: 2026-06-02
updated: 2026-06-02
tags: [project, b-kurs-home, security, supabase, rls, anon-key, research]
related:
  - "[[../b-kurs-home]]"
  - "[[../decisions/decision-no-env-commit-use-example]]"
  - "[[../task-logs/task-log-security-anon-key-rls-issue-11]]"
---

# Supabase anon key 노출 / RLS 점검 (Issue #11)

> ⚠️ 본 노트에 실제 키 값은 기록하지 않는다. 변수명/키 종류/노출 위치만 다룬다.

## 1. 노출 여부 — 확인 결과

| 항목 | 결과 |
| --- | --- |
| 원본 `camshf-crypto/b-kurs` 가시성 | **PUBLIC** |
| 원본 `.env` 추적 여부 | 커밋됨 — `VITE_BKURS_SUPABASE_URL`, `VITE_BKURS_SUPABASE_ANON_KEY` 2개 |
| 노출 키 종류 | anon/**publishable** key (`sb_publishable_` prefix) |
| `service_role` / 서버 비밀 노출 | **없음** (`.env` 2줄뿐, code search `service_role`/`SUPABASE_SERVICE` 결과 없음) |
| `b-kurs-home` history 내 키 존재 | **없음** (`git log --all -S` empty) |
| `b-kurs-home` working tree 내 키 | **없음** (grep empty) |
| 배포 번들 노출 | `VITE_` 변수라 빌드 시 클라이언트 번들에 인라인 → 본질적으로 공개 |

근거 명령:
- `gh repo view camshf-crypto/b-kurs --json isPrivate,visibility` → PUBLIC
- `gh api repos/camshf-crypto/b-kurs/contents/.env` (값은 redact 후 확인)
- `git log --all -S 'sb_publishable'` (b-kurs-home) → empty
- `grep -rI 'sb_publishable|qjcuhczksmouiuhzblcy' .` (b-kurs-home, node_modules 제외) → empty

## 2. 사용 표면

- `src/lib/supabaseContact.ts` — anon key 로 `createClient`.
- `src/pages/contact.tsx` — `supabaseContact.from('contact_inquiries').insert({...})`.
  **insert 만** 수행. select/update/delete 없음.
- 따라서 anon role 의 필요 권한 = `contact_inquiries` INSERT 뿐.

## 3. 위험도 판단

- anon/publishable key 는 **브라우저 노출 전제 키**다. 공개 자체가 설계 의도이고,
  실제 접근 통제는 키 비밀성이 아니라 **RLS** 가 담당한다.
- 고위험 비밀(`service_role`)은 노출되지 않았다 → **위험도 낮음**.
- 단, RLS 가 미설정/오설정이면 공개된 anon key 로 `contact_inquiries` 의
  무단 read/write 가 가능해진다 → 이 경우만 실질 위험.

## 4. 회전 필요 여부

- **결론: 회전 우선순위 낮음 (선택).**
  - publishable key 는 교체본도 공개라 회전의 보안 이득이 제한적.
  - 회전 시 Vercel/로컬 env 갱신 비용 발생.
- **회전 권장 조건**: RLS 가 미설정/오설정으로 확인될 때(방어적 위생), 또는
  운영팀이 노출 위생 차원에서 결정할 때. 회전보다 **RLS 점검이 우선**.

## 5. RLS 점검 — 필요 상태 (라이브 검증 보류)

목표 정책 (`contact_inquiries`):
- RLS **활성화**.
- anon role: **INSERT 만 허용**.
- anon role: SELECT / UPDATE / DELETE **차단**.
- service_role 만 운영/조회용 접근.

> **라이브 검증 보류 사유**: Supabase 대시보드/프로젝트 접근 권한이 없어
> (코드 레포 단독) 실제 RLS 활성화 여부를 확인할 수 없다. operator 의
> 대시보드 접근이 필요하다 (ACCESS). 본 노트는 "필요 상태"만 규정하고
> 실측은 보류 — **검증 못함**으로 명시한다. 결과는 확인 후 본 노트 §5 와
> Issue #11 코멘트에 추가한다.

## 6. 조치 / 산출물

- 코드 레포(PR, Issue #11): env 보안 정책 명문화 — `.env.example` / `README` /
  `AGENTS` / `CLAUDE` 에 (실값 비커밋 / anon=공개값 / RLS 통제 / service_role 금지 /
  `contact_inquiries` INSERT-only) 기재. **실 값/회전/RLS 토글 등 라이브 변경은 미포함.**
- operator 작업(보류): ① 대시보드에서 RLS 실측, ② 필요 시 anon key 회전 + env 갱신,
  ③ 결과 Issue #11 코멘트 + 본 노트 반영.
