---
title: "Secret 관리 정책"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T22:35:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, secret, ops, security]
related:
  - "[[../bkurs-be]]"
  - "[[../external-services-accounts]]"
home_hub: bkurs-be
---

# Secret 관리 정책

## 결론

1. **local**: `.env.local` (gitignore) 또는 shell env. 평문 default 없음.
2. **CI**: ephemeral throwaway 비번 (service container 격리) 또는 GitHub Secrets.
3. **staging / production**: AWS SSM Parameter Store (`/bkurs/{env}/*`) 또는 HashiCorp Vault.
4. **commit 절대 금지**: 평문 비번 / API key / JWT secret. PR 검토 + gitleaks pre-commit.
5. **rotation**: API key 6개월, DB password 1년, JWT secret 변경 시 토큰 무효화.

## 적용 매트릭스

| Secret | local | CI | prod |
|---|---|---|---|
| `DB_PASSWORD` | `.env.local` | `CI_DB_PASSWORD` (ephemeral) | SSM `/bkurs/prod/db/password` |
| `JWT_SECRET` | `.env.local` | GitHub Secrets | SSM `/bkurs/prod/jwt/secret` |
| `ANTHROPIC_API_KEY` (B19+) | `.env.local` | (테스트 안 함) | SSM `/bkurs/prod/anthropic/api-key` |
| `SENDGRID_API_KEY` (B18+) | `.env.local` | (테스트 안 함) | SSM `/bkurs/prod/sendgrid/api-key` |
| `S3_*` (B17+) | `.env.local` (또는 LocalStack) | (테스트 안 함) | SSM `/bkurs/prod/aws/*` |

## 누출 대응 (post-mortem)

| 단계 | 액션 | 시한 |
|---|---|---|
| 감지 | gitleaks scan / 모니터링 알림 | 즉시 |
| 차단 | secret rotate (이미 노출된 secret 무효화) | 1h 이내 |
| 영향 점검 | 로그 / audit (해당 secret 사용 흔적) | 24h |
| repo 정리 (선택) | `git filter-repo` rewrite + force-push (협업자 동시 합의 후) | 1주 |
| 회수 노트 | troubleshooting ledger + 본 vault 결정 노트 update | 1주 |

## 4 대안 — git history rewrite vs rotate

### A. 비번 rotate 만 (rewrite 안 함) — **채택 (현재)**

- **장점**: 협업자 영향 없음. 28 PR stacked 의 base hash 안 깨짐.
- **단점**: history 에 평문 비번 흔적 남음. 단, rotate 된 값이라 무효.
- **채택**: 현재 상황 (대규모 PR stack) 에 안전.

### B. `git filter-repo` 로 rewrite + force-push

- **장점**: history 완전 cleanup.
- **단점**: 28 PR base hash 변경 → 협업자 / GitHub UI 의 모든 stacked PR 깨짐.
- **기각**: launch 전 1회 cleanup 시점에 재검토.

### C. BFG repo cleaner

- B 와 동일 — force-push 필요. 기각.

### D. rotate + history 그대로 + 정기 보안 audit

- 본 안 + 분기 1회 audit 추가. 점진적 안전.
- **future**: 채택 검토 (CI 에 gitleaks workflow 추가).

## 도구 (후속)

- **gitleaks** — pre-commit hook 으로 secret 감지.
- **AWS SSM Parameter Store** — production secret 저장.
- **External Secrets Operator** (k8s) — SSM ↔ ConfigMap 자동 sync.
- **Vault** (자체 호스팅) — 대안.

## 관련

- [[../external-services-accounts]] — 외부 SaaS 계정 별 필요 env
- [[task-log-secret-cleanup]] — 본 작업 task-log
- 후속: gitleaks workflow 추가 / AWS SSM tree 설계
