---
title: "troubleshooting · Agent infinite loop"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, llm, loop, termination]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]", "[[hallucination-loop]]"]
---

# Agent 가 같은 도구 호출 반복 — termination 부재

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

- Agent 가 같은 `ls` / `grep` / `read_file` 무한 반복
- 100+ tool call 후 결국 task 미완료
- token 폭증 / cost 폭증

## 원인

1. Termination 조건 부재 — agent 가 "done" 신호 없음
2. Hallucinated success — agent 가 작업 완료라 믿는데 실제 X
3. Loop in tool results — 같은 search 가 항상 같은 결과
4. 사용자 prompt 가 모호 — agent 가 어디까지 해야 할지 모름

## 해결

### Max iterations
```python
async def run_agent(task, max_steps=20):
    steps = 0
    while steps < max_steps:
        response = await llm.call(...)
        if response.stop_reason == "end_turn":
            return response
        steps += 1
    raise AgentTimeout(f"exceeded {max_steps} steps")
```

→ hard limit. 단 진짜 큰 task 도 cap.

### Progress detection — diff 검증
```python
prev_state = None
no_progress_count = 0

for step in range(max_steps):
    state = current_state()    # file list / test pass / metrics
    if state == prev_state:
        no_progress_count += 1
        if no_progress_count >= 3:
            return ask_user("3 step 동안 진전 없음. 중단?")
    else:
        no_progress_count = 0
    prev_state = state
```

### Termination signal — explicit
```python
# System prompt
"task 완료 시 명시적으로 'TASK_COMPLETE' 라고 응답하세요.
3 step 동안 진전 없으면 'NO_PROGRESS' 응답."

# Loop
if "TASK_COMPLETE" in response.content:
    return success
if "NO_PROGRESS" in response.content:
    return ask_user(...)
```

### Same-tool-same-args detection
```python
recent_calls = deque(maxlen=5)

for step in steps:
    if (tool, args) in recent_calls and recent_calls.count((tool, args)) >= 3:
        # 같은 call 3번 → 의심
        inject_message("같은 도구 같은 args 3회 반복. 다른 접근 필요.")
    recent_calls.append((tool, args))
```

### Sub-task 분할 — 사용자 의도 명시
```
Bad prompt: "fix all bugs"   ← 무한히 가능
Good prompt: "fix the OOM bug in OrderService.process(). PR 생성 시 종료."
```

명확한 done criteria.

### Tool result 의 결정적 변화
```python
# Bad — tool 가 매번 same result
search(query="bug")   # 항상 같은 list

# Good — pagination / filter
search(query="bug", page=2)
search(query="bug", filter={"status": "open"})
```

→ tool result 의 정보량이 누적되도록 design.

## Yule Studio Agent

`troubleshooting_enforcer` 가 같은 signature 의 troubleshooting 2회 이상 → mistake_ledger 자동 승격. 같은 loop pattern 도 추적.

```python
if loop_detector.detect(history):
    troubleshooting_ledger.append(TroubleshootingRecord(
        signature="agent-loop::" + tool_pattern,
        category=Category.LOOP,
        outcome=Outcome.STOPPED,
    ))
```

## 진단

```python
# Step 별 tool / args 추적
for step in trace:
    print(f"#{step.idx} {step.tool}({step.args})")

# 가장 자주 호출된 tool
Counter(s.tool for s in trace).most_common(5)
```

## 재발 방지

- max_steps 표준 (20-50)
- progress detection (state diff)
- tool call dedup (recent N step 의 same call 차단)
- 사용자 prompt 의 done criteria 명시
- 모니터링: avg steps per task

## 관련

- [[ai-agent]] / [[hallucination-loop]] / [[multi-turn-forgetting]]
