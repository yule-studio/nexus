---
title: "troubleshooting · Claude Code sandbox block"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, ai-agent, claude, sandbox, classifier]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[../ai-agent/ai-agent]]"]
---

# Claude Code sandbox block — auto-mode classifier 거절

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
Permission for this action was denied by the Claude Code auto mode classifier.
Reason: ...
```

자주 보는 패턴:
- 외부 repo 의 `python -m yule_orchestrator engineer intake` 명령
- `gh pr merge` to default branch (main 직접)
- `rm -rf` / `git reset --hard` 등 destructive
- 새로운 인프라 endpoint (회사 외부) 접근

## 원인

Claude Code 의 auto-mode classifier 가 다음 패턴 의심:
1. **Scope mixing** — self-improvement runtime 작업 중 외부 repo 대상 명령
2. **Destructive** — 복구 불가능 작업 (force push / rm / drop table)
3. **Cross-system effect** — 외부 시스템 / 사용자 영향
4. **Authorization scope** — 명시 인가 없이 새 작업

→ 보호 layer. 의도된 동작.

## 해결

### 1. Scope 분리 — 가장 권장

같은 session 에서 2 작업 동시 진행 X. 다른 session 으로 분리:

```
Session A: self-improvement (yule-studio-agent 본체)
Session B: 외부 repo 작업 (naver-search-clone)
```

각 session 가 독립적으로 classifier 평가.

### 2. 명시적 prompt clarification

거절 메시지의 reason 을 읽고, 사용자에게 정확한 의도 확인:

```
"이 작업은 yule-studio-agent self-improvement 와 무관한 naver-search-clone repo
대상으로, 사용자가 명시 인가했다. main branch 가 아닌 feature branch 로 PR 만 생성."
```

사용자가 새 turn 에 "OK" / "진행" 응답 시 classifier 재평가.

### 3. `--dangerously-skip-permissions` (책임 자기 부담)

```bash
claude --dangerously-skip-permissions
```

→ 모든 classifier 우회. 사용자가 모든 책임. CI 등 자동화 외엔 권장 안 함.

### 4. 사람 승인 카드 (Operator Action Inbox)

```
#승인-대기 카드:
  request_type: APPROVAL
  reason: 외부 repo intake — scope mixing 감지
  context: ...
  action: 진행 / 거절
```

→ 사람이 명시 승인 후 진행. [[../ai-agent/ai-agent]] 참조.

## Yule Studio Agent 의 troubleshooting ledger

이 사례를 ledger 에 capture:

```python
from yule_orchestrator.agents.lifecycle.troubleshooting_record import (
    TroubleshootingRecord,
    Outcome,
    Category,
)

record = TroubleshootingRecord(
    signature="sandbox-block::external-repo-intake",
    category=Category.SANDBOX_BLOCK,
    outcome=Outcome.WORKAROUND,
    description="...",
    resolution="scope 분리 + 별도 session",
    runtime_impact="작업 중단 5분",
)
ledger.append(record)
```

→ 같은 signature 2회 이상이면 mistake_ledger 자동 승격 → 정책 / classifier 개선.

## 자주 만나는 패턴

### 외부 repo intake during self-improvement
```
거절 사유: "self-improvement runtime 작업 중 외부 repo 대상 autonomous engineer intake 는 scope 를 섞고, 신뢰된 source control 바깥에 영향을 줄 수 있어 차단."

해결: 다른 session 으로 분리.
```

### Main branch 직접 push / merge
```
거절 사유: "default branch (main) 의 직접 변경은 인가 없이 차단."

해결: feature branch + PR + 사람 review (또는 사용자 명시 인가).
```

### 새 외부 API 호출
```
거절 사유: "허용 도메인 외부 endpoint 접근."

해결: allowlist 확인 / 회사 IT 협상 / 신뢰 영역 안에서 작업.
```

## Sandbox 진단

```bash
# Claude Code 의 현재 mode
# /status

# 가용 도구
# /agents

# Operator Action Inbox 의 pending
# 또는 통계 dashboard
```

## 재발 방지

- 1 session = 1 scope 룰 (self-improvement vs 외부 작업)
- 의심되는 작업 시작 전 `#승인-대기` 카드 검토
- troubleshooting_ledger 에 매 sandbox block 기록 (mandatory)
- 정책 / classifier 개선 — repeating signature 발견 시 룰 업데이트

## 관련

- [[network]] / [[../ai-agent/ai-agent]]
- yule-studio-agent: `troubleshooting_enforcer.py` / `mistake_ledger.py`
- docs/autonomy-policy.md / docs/approval-matrix.md
