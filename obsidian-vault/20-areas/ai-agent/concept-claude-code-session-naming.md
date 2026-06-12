---
title: "Claude Code 세션 네이밍 / 수명주기 관리 규칙"
kind: knowledge
project: ai-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-12T18:00:00+09:00
tags: [area, ai-agent, claude-code, workflow, session-management]
home_hub: areas
related:
  - "[[ai-agent]]"
  - "[[agent-runtime/agent-runtime]]"
---

# Claude Code 세션 네이밍 / 수명주기 관리 규칙

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1 | 2026-06-12 | engineering-agent/tech-lead | 최초 작성. `/resume` 비대 문제 → 분류·네이밍 규칙 정립 |

## 핵심 요약

Claude Code 의 `/resume` 목록은 세션 `.jsonl` 이 누적되면서 비대해진다 (한 프로젝트에 34개 누적, 최대 196MB 단일 세션 관측). 세션 제목이 "첫 user 프롬프트 앞부분" 으로만 표시되어 무엇이 끝났고 무엇이 진행 중인지 구분이 어렵다. 이를 막기 위해 **세션을 3등급으로 분류**하고, **첫 메시지 첫 줄을 의도적 제목으로** 쓰는 네이밍 규칙을 둔다.

## 내 해석 (왜 이 규칙이 필요한가)

- **왜 필요한가** — `/resume` 는 "이어서 할 일" 을 고르는 곳인데, 끝난 작업까지 섞이면 재개 대상을 찾는 비용이 매 세션 누적된다. 세션 제목이 프롬프트 앞 160자라 동일 프리픽스("공통 작업 규칙: ...")가 여러 세션에서 똑같이 보여 식별 불가.
- **안 하면 생기는 문제** — (1) 끝난 세션을 실수로 재개해 중복 작업, (2) 진짜 진행 중 세션이 묻혀 누락, (3) 디스크 수백 MB~GB 누적, (4) 중복/빈 세션이 영원히 남음.
- **대안과 트레이드오프** — 무지성 전체 삭제는 빠르지만 작업 맥락(결정/후속)이 사라진다. 그래서 **요약은 남기고(archive) 원본 jsonl 은 보존**하되 resume 관리에서만 빼는 절충을 택한다. 요약 SSoT 는 코드 레포 쪽 `docs/claude-session-archive.md`.

## 적용 맥락

### 1. 세션 3등급 분류

| 등급 | 정의 | 처리 |
| --- | --- | --- |
| **KEEP** | 이어서 작업해야 하는 진행 중 세션 | `/resume` 로 계속 관리 |
| **ARCHIVE** | 작업은 끝났고 요약만 남기면 되는 세션 | `docs/claude-session-archive.md` 에 요약 후 resume 관리 제외 |
| **DELETE_CANDIDATE** | `/clear` 직후 빈 세션, 2KB 이하 실험, 중복(동일 프롬프트·동일 시간대) 세션 | 목록만 제안 → **사람 승인 후** 실삭제 (자동 삭제 금지) |

분류 신호:
- **빈/실험 세션** — user 메시지 0개, 파일 2KB 이하, 첫 프롬프트 없음 → DELETE_CANDIDATE.
- **중복 세션** — 동일 첫 프롬프트 + 같은 분 단위 시작 시각 + 같은 브랜치 → 한 개만 archive, 나머지 DELETE_CANDIDATE.
- **병렬 fan-out 의심** — 동일 프롬프트로 동시에 시작했으나 내용이 분기한 클러스터 → "review before delete" 로 보류.
- **진행 중** — 최근 활동(mtime)이 며칠 내 + 미완 작업 → KEEP.

### 2. 세션 네이밍 규칙 (앞으로)

Claude Code 는 세션 제목을 따로 못 붙이므로 **첫 user 메시지 첫 줄을 제목처럼** 쓴다.

```
[<repo>] <type>/<issue-or-branch-anchor> — <한 줄 목적>
```

- `<repo>` — `yule-studio-agent` / `bkurs-fe` / `yule-agent-vault` 등 작업 대상 레포.
- `<type>` — `feat` / `fix` / `chore` / `refactor` / `docs` / `vault` / `research`.
- `<issue-or-branch-anchor>` — 이슈 번호(`#185`) 또는 브랜치 anchor. governance 의 "issue 번호를 branch name 에 anchor" 규칙과 일치.
- `<한 줄 목적>` — 끝나고 봐도 식별되는 구체 목적. "공통 작업 규칙:" 같은 공통 프리픽스로 시작 금지.

예시:
- `[yule-studio-agent] refactor/#185 — commands/__init__ facade 슬림화`
- `[bkurs-fe] feat/signup — 학생 회원가입 inroad-high 구조 최신화`
- `[yule-agent-vault] vault/aws — AWS 서비스 카테고리 사전 작성`

### 3. 정리 주기

- 세션이 ~20개를 넘으면 1회 분류 sweep.
- ARCHIVE 한 세션은 jsonl 은 두되 요약을 archive 문서에 추가.
- DELETE_CANDIDATE 는 모아서 사람에게 일괄 승인 요청 (파괴적 작업 사람 승인 필수 규칙 준수).

## 관련 노트

- [[ai-agent]] — LLM 에이전트 운영 영역 Hub
- [[agent-runtime/agent-runtime]] — 에이전트 런타임 영역
- 요약 SSoT: `yule-studio-agent/docs/claude-session-archive.md`

## 참고

- 세션 원본 경로: `~/.claude/projects/<encoded-project-path>/<session-id>.jsonl`
- 분류 근거: 2026-06-12 yule-studio-agent 34개 세션 sweep (KEEP 3 / ARCHIVE 25 / DELETE_CANDIDATE 6).
- governance 연계: branch prefix·issue anchor 규칙은 `docs/engineering-agent-governance.md`, `policies/reference/BRANCH_STRATEGY.md`.
