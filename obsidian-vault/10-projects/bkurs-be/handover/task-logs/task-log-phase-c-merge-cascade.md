---
title: "Phase C task-log — dev merge cascade (28 PR 일괄 반영)"
kind: task-log
project: bkurs-be
status: completed
created_at: 2026-05-20T23:20:00+09:00
agent: engineering-agent/coding
tags: [task-log, phase-c, merge, ops]
related:
  - "[[../bkurs-be]]"
  - "[[task-log-phase-b-abstractions]]"
home_hub: bkurs-be
---

# Phase C — Merge cascade

## 결과

- dev branch ← 모든 Phase A/B stack 반영 (28 PR 분량).
- merge commit: `941ebaf :rocket: dev ← Phase A/B 전체 stack 머지 (#4~#76, 28 PR)`.
- 35 open PR close + 37 issue close + 33 feature branch 삭제.
- 원격 branch 3개만 잔존 (main / dev / HEAD ref).

## 절차

### 1. 로컬 머지

```bash
git checkout dev
git pull origin dev
git merge origin/feature/issue-75-b21-pdf --no-ff
git push origin dev
```

→ dev 가 최상위 stack tip 의 모든 commit 흡수 (B2A 부터 Phase B 까지).

### 2. PR 일괄 close

GitHub `gh pr edit --base dev` 가 "no new commits between base and head" 로 실패 →
모든 commit 이 이미 dev 에 있다는 의미. retarget 대신 close + delete-branch.

```bash
for pr in 8 10 12 ... 76; do
  gh pr close $pr --comment "merged via dev cascade — commit 941ebaf" --delete-branch
done
```

35 PR 일괄 close + branch 삭제.

### 3. Issue 일괄 close

PR close 만으로는 issue 가 close 안 됨 (PR 의 `Closes #N` 은 merge 시점에만 발동).

```bash
for issue in 3 5 7 ... 75; do
  gh issue close $issue --comment "완료 — dev cascade merge 941ebaf"
done
```

37 issue close.

## 머지된 commit 군 (dev 기준 cumulative)

| Phase | PR# | 내용 |
|---|---|---|
| B2A | #4 (이전 머지) | Account + JWT |
| B2B-D | #6 #8 | Refresh / Cookie / Academy Connect |
| B3A-C | #10 #12 #14 | Academy CRUD / Teacher invite / Announcement |
| B4A-D | #16 #18 #20 #22 | Academy info / Plan / Payment Gateway (Mock) / Owner transfer |
| B5 | #24 | Student profile |
| B6A-C | #26 #28 #30 | Course / Session+Attendance / Teacher permission |
| B7A | #32 | Homework |
| B7B | #34 | PersonalStatement |
| REFACTOR | #36 | AbstractIntegrationTest 추출 (-1270 lines) |
| B7C-I | #38 #40 #42 #44 #46 #48 #50 | Question / MockInterview / StudentRecord / Discussion / ReadingNote / PerformanceEvaluation / Simulation |
| B9 | #52 | Auto-bridge (PE/Disc/Sim/Reading → record) |
| B10 | #54 | 통합 검색 (5 도메인 LIKE) |
| B11 | #56 | 학생 통계 (7 영역) |
| B13 | #58 | 강사 dashboard (수업별) |
| B16 | #60 | 학원장 dashboard (학원 rollup) |
| B12 | #62 | 학기 자동 결정 (한국 학년도) |
| A-1 | #64 | Secret cleanup (env var 화) |
| A-2 | #66 | Common Response Envelope foundation |
| B17 | #68 | FileStorageService (presigned URL) |
| B18 | #70 | NotificationService (이메일/SMS) |
| B19 | #72 | AiAssistant (Claude 추상화) |
| B20 | #74 | SttService (Whisper/Clova 추상화) |
| B21 | #76 | PdfGenerator + 생기부 PDF endpoint |

## 누적 통계 (dev 시점)

- **총 commit**: ~100+ (dev 기준)
- **endpoint**: ~165 (153 + 신규 5 file + 3 pdf + 4 dashboard/stats)
- **회귀**: ~295 GREEN (전체 도메인 + abstractions + envelope foundation)
- **migration**: V1~V23 (23 개)
- **외부 서비스 abstraction**: 6 (Payment / FileStorage / Notification / Ai / Stt / Pdf)
- **vault docs**: 14+ (architecture / accounts / gap / 5 phase task-log / decisions)

## 후속 (다음 작업)

### Production 진행

- **B22~B29 real plug-in** (외부 SaaS 실 호출)
- **dev → main release PR** (정식 launch 시)
- **company-infra repo 분리** (docker-compose / IaC)

### 코드 품질

- **ResponseEnvelopeMigration PR** — 153 기존 endpoint envelope 전환
- **@Query COUNT/AVG 최적화** (B11/B13/B16 stats N+1)
- **FULLTEXT + ngram** (B10 검색 한글)

### bkurs-fe 작업

- **inroad-high 신규 화면 분석 + FE 구현** (task #178)
- **Supabase → Spring 전환** (5 절차, vault `external-services-accounts §0`)

## 관련

- [[task-log-phase-b-abstractions]] — Phase B 5 abstraction 통합
- [[../hybrid-external-services]] — 아키텍처 + Mermaid
- [[../services-accounts]] — 외부 SaaS 계정 + 비용 + Supabase 매핑
- [[../fe-be-gap]] — FE vs BE gap
- [[../bkurs-be]] — 진행 로그 hub
