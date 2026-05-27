---
title: "troubleshooting · Multi-turn 망각"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, memory, context, summarization]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]", "[[context-overflow]]"]
---

# Agent 가 직전 결정 잊음

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

- Turn 20 에서 결정한 사항을 turn 50 에서 다시 묻거나 반대로 결정
- Compaction 후 옛 결정 / 제약 사라짐
- 같은 작업 중복

## 원인

- Context 가 길어지며 옛 turn 의 중요 결정이 attention 약함
- Summarization (auto-compact) 가 핵심 결정 빠뜨림
- Sliding window 가 핵심 turn drop

## 해결

### Sticky decision log (system message 에 누적)
```python
decisions = []   # 세션 내 중요 결정 누적

def add_decision(decision: str):
    decisions.append(decision)

# 매 turn 의 system message
def build_system():
    base = "You are a helpful assistant."
    if decisions:
        base += "\n\n## 이 세션의 결정:\n" + "\n".join(f"- {d}" for d in decisions)
    return base
```

→ 결정 사항이 매 turn 의 system 에 (cache_control 적용).

### Explicit recap
```python
# Long conversation 의 중간 / 끝에
recap = await llm.summarize_decisions(messages)
messages.append({"role": "user", "content": f"진행 상황 recap:\n{recap}\n계속 진행하세요."})
```

### TaskList / TodoList — 외부 영속
Claude Code 의 `TaskCreate` / `TaskList` — 작업 상태가 file 에 영속:
```python
tasks = [
    {"id": 1, "status": "completed", "content": "user auth 추가"},
    {"id": 2, "status": "in_progress", "content": "endpoint 보안 강화"},
    {"id": 3, "status": "pending", "content": "테스트"},
]
```

→ context 변화와 무관하게 task 상태 유지.

### Memory file (외부 영속)
```python
# /Users/.../memory/MEMORY.md 에 누적
- 사용자 선호: 한국어 응답
- 결정: PR 당 최소 3 commit
- 진행 중: bkurs-fe re-implementation
```

→ Claude Code 의 auto-memory. 세션 간 유지.

### Compaction 시 명시적 보호
```python
# Auto-compact 가 옛 turn 요약할 때
preserved = []
for msg in messages:
    if "DECISION:" in msg.content or "REMEMBER:" in msg.content:
        preserved.append(msg)

summary = await summarize(messages_to_compact)
new_messages = [
    {"role": "system", "content": f"이전 요약: {summary}"},
    *preserved,
    *recent_messages,
]
```

### 사용자가 명시적으로 reminder
```
"Remember: 우리는 N+1 문제 해결 중. JPA fetch join 패턴 적용."
```

→ "Remember" prefix 가 agent 의 attention 강화.

## Anti-pattern

```
Bad: 모든 turn 을 동등하게 → 옛 결정 잊음
Bad: 옛 turn 완전 drop → 맥락 잃음
Bad: 매번 사용자에게 다시 물어봄 → UX 나쁨
```

## Yule Studio Agent 의 session memory

```python
# session.extra 에 영속
{
  "work_mode": "approval_required",
  "topology": "single_repo",
  "scope": "spring-only",
  "decisions": [...],
  "blockers": [...],
}
```

→ session 복구 시 위 dict 가 system context 로.

## 진단

```python
# 같은 질문 / 같은 결정 반복 detection
recent_questions = [extract_intent(m) for m in messages[-20:]]
duplicates = [q for q in recent_questions if recent_questions.count(q) > 1]
if duplicates:
    logger.warn("possible_forgetting", duplicates=duplicates)
```

## 재발 방지

- Decisions / constraints 는 system message 에 sticky
- TaskList / TodoList 활용 (외부 영속)
- Memory file (`MEMORY.md`) 활용
- Compaction 가 중요 결정 보존하도록 설계
- 사용자에게 "Remember: ..." 패턴 prompt template

## 관련

- [[ai-agent]] / [[context-overflow]] / [[agent-infinite-loop]]
