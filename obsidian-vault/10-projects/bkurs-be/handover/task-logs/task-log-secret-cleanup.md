---
title: "Secret cleanup task-log — env var 화 + prod profile"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T22:35:00+09:00
agent: engineering-agent/coding
tags: [task-log, secret, ops, security]
related:
  - "[[../bkurs-be]]"
  - "[[../services-accounts]]"
  - "[[../hybrid-external-services]]"
home_hub: bkurs-be
---

# Secret cleanup task-log

## 결과

- 이슈: #63
- PR: #64 (draft, stacked on #62 B12)
- 브랜치: `chore/issue-63-secret-cleanup`
- 1 commit (cleanup)
- 로컬 test PASS (`DB_PASSWORD` env 주입)

## 변경 파일

| 파일 | 변경 |
|---|---|
| `docker-compose.yml` | `${MYSQL_PASSWORD:?...}` env var 필수 |
| `application-local.yml` | `${DB_PASSWORD:?...}` env var 필수 (default 없음) |
| `application-prod.yml` | 신규 — 모든 secret env, SSL/HTTPS, stacktrace 숨김 |
| `.env.example` | 신규 — DB / JWT / 외부 SaaS (B17~) 가이드 |
| `.github/workflows/ci.yml` | CI 격리 throwaway 비번 (ephemeral service) |
| `README.md` | `.env.local` 가이드 + secret 정책 섹션 |

## Production profile 보강

`application-prod.yml` 에 추가된 안전장치:
- `${DB_HOST:?required}` — env 누락 시 startup fail (silent default 회피)
- `requireSSL=true&verifyServerCertificate=true` — MySQL TLS 강제
- `forward-headers-strategy: native` — reverse proxy (ALB/nginx) 뒤 IP 정상화
- `server.error.include-stacktrace: never` + `include-message: never` — 5xx 에 내부 정보 노출 금지
- `format_sql: false` — 운영 log 노이즈 감소
- `file.name: /var/log/bkurs-be/app.log` — 컨테이너 외 영속 logging

## 안전 검증

- ✅ 모든 `13950991` 평문 잔존 제거 (`grep -rn "13950991"` 0 hit)
- ✅ 로컬 `./gradlew test --tests AuthApiTest` PASS (env override 동작)
- ✅ CI throwaway 비번은 ephemeral GitHub Actions service container 만 — 외부 노출 없음

## 후속 운영 action (개발 외)

1. **DB user `mw` 비번 rotate** — 현재 공개된 `13950991*` 무효화. 로컬 MySQL 사용 중인 모든 개발자 동시 진행.
2. **git history rewrite (선택)** — `git filter-repo --replace-text` 로 commit 3개 (`e39c560`, `777eb2e`, `4138a18`) 정리. force-push 필요 → 협업자 영향 → 비번 rotate 만 으로 충분 (이미 무효).
3. **production secret manager 구축** — AWS SSM Parameter Store tree `/bkurs/prod/*` 또는 Vault.

## 선택 이유 — handover

### A. local default 제거 (`${DB_PASSWORD:?...}`)

- **선택 이유**: 평문 default 가 다시 commit 되는 사고 방지. 사용자가 `.env.local` 명시적으로 만들어야 부팅 가능.
- **기각된 대안**: `${DB_PASSWORD:13950991*}` (default 평문 유지) — 같은 보안 사고 재발.
- **trade-off**: 신규 개발자 1단계 추가 (`.env.local` 셋업) — README 가이드로 보완.

### B. CI throwaway 비번 (rotate 아님)

- **선택 이유**: CI 는 매 run 마다 격리된 service container — 영구 secret 의미 없음.
- **기각된 대안**: GitHub Secrets 사용 — 오버킬 (CI 내부 격리 service 용).
- **trade-off**: 공개 yaml 에 비번 같은 문자열 있지만, 외부 접근 불가능한 ephemeral 환경.

### C. application-prod.yml 별도 (env var 누락 시 startup fail)

- **선택 이유**: silent default fall-back 으로 production 보안 약화 방지. `${X:?required}` 가 missing env 일 때 명시적 fail.
- **기각된 대안**: production 도 `application-local.yml` 재사용 + 모든 값 env override 의존 — fall-back 사고 가능.

### D. git history rewrite 보류 (비번 rotate 권장)

- **선택 이유**: rewrite 는 force-push → 협업자 영향. 이미 origin 에 push 된 commit hash 변경 시 다른 PR 들 base 가 깨짐 (28 PR stacked!).
- **대안**: DB 비번 rotate 시 노출된 평문은 무효화됨 — 안전.
- **future**: 정식 launch 전 1회 cleanup repo rebuild 시 검토.

### E. CHANGELOG 누락 의도

- 본 PR 은 chore (secret) — 도메인 feature 아님. CHANGELOG 는 도메인 단위 기록.
- task-log + decision 노트 (본 문서) 로 audit trail 충분.

## 관련

- [[../services-accounts]] §5 — Secret 누수 (현재 상태) 항목
- [[../hybrid-external-services]] §9 — Secret 관리 (modify 대상) 표
- [[../bkurs-be]] — 진행 로그
- 후속: Phase A-2 Common Response Envelope foundation
