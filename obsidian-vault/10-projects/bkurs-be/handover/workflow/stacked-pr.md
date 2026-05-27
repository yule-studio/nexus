---
title: "bkurs-be — Stacked PR 워크플로"
kind: handover
status: stable
created_at: 2026-05-21T22:40:00+09:00
tags: [bkurs-be, handover, workflow, pr, git-flow]
related:
  - "[[../bkurs-be]]"
  - "[[../../../memory/feedback_commit_splitting_policy]]"
  - "[[../../../memory/feedback_pr_issue_labels_assignees]]"
home_hub: bkurs-be
---

# bkurs-be — Stacked PR 워크플로

> "왜 PR 이 #4 → #6 → #8 → ... 순서로 stack 되고, 어디서 dev cascade 가 일어났나" + "Phase
> A/B/C 가 뭐고 사이클 2 는 뭔가". 진행 로그는 [[../bkurs-be]] §6, 본 문서는 **흐름 SSoT**.

## 1. 한 장 요약

```
main (initial Spring base 1 commit, 사용자 직접 push)
 │
 ▼
dev (모든 feature 의 통합 branch)
 │
 ├── feature/issue-1-setup            (B1 — setup) ────────┐
 │   └── PR #2 → dev (merge --delete-branch)              │
 │                                                          │
 ├── feature/issue-3-b2a-account-jwt  (base=dev)            │
 │   └── PR #4 (CI GREEN, 사용자 manual merge)             │
 │       └── feature/issue-5-b2b-refresh-cors (base=B2A)   │ stacked
 │           └── PR #6                                      │
 │               └── feature/issue-7-b2c-academy-connect ...│
 │                   └── PR #8                              │
 │                       └── ... (B3A-B7I-REFACTOR-B9-...)  │
 │                                                          │
 │  Phase A (#64 secret cleanup, #66 envelope foundation)   │
 │  Phase B (#68 file, #70 noti, #72 ai, #74 stt, #76 pdf)  │
 │                                                          │
 │  Phase C 완료 = dev cascade merge (commit 941ebaf)       │
 │    35 PR close + 37 issue close + 33 branch 삭제         │
 │                                                          │
 │  사이클 2 (B22 PortOne / B23 SendGrid / B24 Claude /     │
 │            B25 Clova / restructure / Swagger / Docker /  │
 │            GHCR / dev 배포) — commit 218a89b              │
 │                                                          │
 │  사이클 2 이후 — past-question (V28) / major-question    │
 │  (V29) / grade 1~6 / change-password / membership view ─ │
 │                                                          ▼
                                                          최신 dev
```

## 2. 브랜치 규칙

| 브랜치 | 용도 | 보호 |
| --- | --- | --- |
| `main` | release 직전만. initial Spring base 1 commit 이후 거의 손대지 않음 | 직접 push 금지 (classifier 차단) |
| `dev` | 모든 feature 의 통합 (= integration branch). 사이클 종료 시 main 으로 promote 검토 | direct push 금지, PR only |
| `feature/issue-{n}-{slug}` | 한 issue = 한 feature branch = 한 PR | |
| `chore/issue-{n}-{slug}` | docs / CI / config | |
| `release/*` | (도입 예정) main promote 직전 | |

자세히 `docs/BRANCH_STRATEGY.md` (repo).

## 3. PR 구성 규칙

- **1 PR = 1 issue = 최소 3 commit** (docs-only chore 만 squash 허용)
- commit: 한국어 + gitmoji 1개 + 3 섹션 (변경 이유 / 주요 변경 / 비고). 자세히 `docs/COMMIT_CONVENTION.md`
- assignee: `codwithyc`
- label: `gh label list` vocab 최소 1개 (`enhancement` / `documentation` / `chore` / `bug` ...)
- PR body 5 섹션: 목적 / 범위 / 리스크 / 테스트 / 이슈 linkage + audit block
- author 통일: `codwithyc`

## 4. Stacked PR 의 의미

**base = 이전 PR 의 feature branch.** 이전이 merge 되기 전이라도 다음 작업이 진행 가능 — 의존
chain 이 명시적.

예시:

```
feature/issue-5-b2b-refresh-cors    (base = feature/issue-3-b2a-account-jwt)
  → PR #6 (base = #4 의 branch, NOT dev)
```

머지 흐름:

1. **위에서부터** merge. PR #4 (B2A) → dev. 자동으로 PR #6 (B2B) 의 base 가 dev 로 retarget.
2. PR #6 의 CI 가 dev base 로 재실행.
3. GREEN 시 PR #6 → dev.

**Pitfall**: B2A 머지 전에 B2B 코드를 dev 에 cherry-pick 금지 — base mismatch. 항상 순서대로.

## 5. CI 흐름

`.github/workflows/`:

- **CI** — every push to feature/*, dev. MySQL 9 service + JDK 21 + `./gradlew clean check`. 약 1m43s.
- **image publish** (cycle 2 #90) — `dev` push 시 GHCR 로 Docker image push. tag: `latest` + commit SHA.

## 6. 사이클 = 큰 단위 작업

### cycle 1 (B1 ~ Phase C, 2026-05-19 ~ 2026-05-20)

- B1 setup → B2 auth (A/B/C) → B3 academy (A/B/C) → B4 plan (A/B/C/D) → B5 student → B6 course (A/B/C) → B7 학습 (A~I) + REFACTOR → B9 record bridge → B10 search → B11 stats → B13 teacher dash → B16 owner dash → B12 academic period → Phase A (secret / envelope foundation) → Phase B (file/noti/ai/stt/pdf gateway interfaces) → **Phase C dev cascade merge**
- 결과: ~165 endpoint / ~295 회귀 GREEN / V1~V23 / 6 외부 abstraction

### cycle 2 (real SaaS + 배포, 2026-05-20 ~ 2026-05-21)

- B22 PortOne 실 plug-in (#101)
- B23 SendGrid 실 plug-in (#102)
- B24 Claude 실 plug-in (#103)
- B25 Clova STT 실 plug-in (#104)
- restructure (`web/` → `controller/`, dto split, handler 분리)
- Swagger UI 8 group 한국어
- Dockerfile + GHCR workflow + application-dev.yml + docker-compose.dev.yml
- 결과: **dev 배포 준비 완료** (commit 218a89b)

### cycle 2 이후 잔여 작업 (2026-05-20 ~ 현재)

- past-question (V28, #152) / major-question (V29, #153)
- grade 1~3 → 1~6 (#150)
- change-password (#149)
- student membership view (#151)
- envelope migration #87 (별도 진행)

## 7. dev cascade merge — Phase C

cycle 1 의 stacked chain 이 길어지면서 발생 가능한 위험: 가장 위 PR (#4) 가 머지되기 전에 chain
끝 PR (#62) 의 fix 가 필요하면 base 갱신 비용 큼.

해결: 한 cycle 끝나면 **모든 PR 을 한 번에 dev 로 cascade merge**.

흐름 (commit 941ebaf):

1. 모든 stack 의 코드를 dev 에 직접 적용 (단일 merge commit).
2. 35 PR을 `gh pr close` 로 정리.
3. 37 issue 를 close + 33 branch 삭제 (`gh api -X DELETE`).
4. dev 가 모든 PR 의 합 상태.

> ⚠️ dev cascade 는 **cycle 종료 시점만**. 평시는 1 PR = 1 merge 원칙.

## 8. local 작업 절차 (1 PR 단위)

```bash
# 1. dev 동기화
git checkout dev && git pull origin dev

# 2. 새 issue 생성
gh issue create --title "[Feature] B26 ..." --body "..." \
    --assignee codwithyc --label "enhancement"

# 3. feature branch (base = 이전 stack 의 head 또는 dev)
git checkout -b feature/issue-{n}-{slug}

# 4. 코드 + 테스트 + 도메인 advice + V*.sql
./gradlew clean check    # GREEN 확인

# 5. commit 3개 이상 (한국어 + gitmoji + 3 섹션)
git commit -m ":sparkles: ..."
git commit -m ":white_check_mark: ..."
git commit -m ":memo: ..."

# 6. push + PR
git push -u origin feature/issue-{n}-{slug}
gh pr create --base feature/issue-{prev}-... \
    --title "..." --body "..." \
    --assignee codwithyc --label "enhancement"

# 7. CI GREEN 확인
gh pr checks

# 8. 사용자 manual merge — auto-merge 절대 금지 (memory feedback)
```

## 9. 자동 머지 금지

memory: [[../../../memory/feedback_no_auto_merge]] — self-improvement / delegated operator 경로에서는
CI 성공해도 머지 자동 금지. 항상 draft PR 까지만, 머지는 사용자.

예외: [[../../../memory/feedback_auto_merge_authorization]] 의 F1~F8 cycle 한정 self-merge — bkurs-be
에는 미적용.

## 10. vault 기록 정책

memory: [[../../../memory/feedback_vault_project_tracking]] — PR 별로 본 hub + 20-areas 보강 필수.
- 10-projects/bkurs-be/{task-logs,decisions,research}/ 누적
- 20-areas/backend/spring/... 의 기존 노트 보강 또는 신규 추가
- PR 머지 전 vault 노트 1회 이상 업데이트

## 11. 회수 / pitfall 사례

| 회수 | 원인 | 해결 |
| --- | --- | --- |
| `git push` 차단 | classifier 가 main / dev 직접 push 차단 | initial commit 1개만 사용자 직접 push, 그 후 PR |
| stack base 잘못 — dev 에 머지된 PR 의 branch | 이전 PR merge --delete-branch 했더니 다음 PR base 사라짐 | 다음 PR 의 base 를 dev 로 retarget |
| save vs saveAndFlush | random 32-char 충돌 시 재시도 (B3A) | `saveAndFlush` 로 즉시 unique violation 받기 |
| ERROR dispatch `/error` 401 | B4A | `GlobalValidationExceptionHandler` 추가 |

## cross-link

- [[../bkurs-be]] — 진행 로그 (PR / cycle 별)
- `docs/BRANCH_STRATEGY.md` (repo)
- `docs/COMMIT_CONVENTION.md` (repo)
- [[../../../memory/feedback_commit_format]]
- [[../../../memory/feedback_commit_splitting_policy]]
- [[../../../memory/feedback_no_auto_merge]]
- [[../../../memory/feedback_pr_issue_labels_assignees]]
- [[../../../memory/feedback_vault_project_tracking]]
