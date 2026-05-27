---
title: "troubleshooting · LLM context overflow"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, llm, claude, context, token]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# Context overflow — `messages length exceeds limit`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
Error: messages length exceeds limit of 200000 tokens
Error: input_tokens (220000) exceeds the maximum
```

- 긴 conversation 후 발생
- 큰 tool result 1회로도
- 다중 file 첨부 시

## 원인

LLM 의 context window 한도 도달:
- Claude Sonnet 4.6 — 200K tokens
- Claude Opus 4.7 — 200K (또는 1M context 변형)
- system + history + 현재 message + tools 가 합쳐 한도 초과

## 해결

### Summarization (요약)
```python
# 옛 turn 을 요약 + drop
if estimated_tokens > 150_000:
    summary = await llm.summarize(history[:-10])
    history = [{"role": "system", "content": f"이전 대화 요약: {summary}"}] + history[-10:]
```

### Sliding window
```python
MAX_TURNS = 50
recent = history[-MAX_TURNS:]
```

→ 단순. 단 중요한 옛 결정 잃을 위험.

### Sub-agent 분리
큰 task 를 sub-agent 에 위임. sub-agent 결과만 main context 로 return.

```python
result = await spawn_subagent(
    role="researcher",
    task="검색 + 요약",
)
# main context 에는 result.summary 만 추가 (full conversation 안 추가)
```

### `cache_control` (Anthropic prompt caching)
```python
messages = [
    {"role": "user", "content": [
        {"type": "text", "text": large_static_doc, "cache_control": {"type": "ephemeral"}},
        {"type": "text", "text": user_question},
    ]},
]
```

→ static 부분 (system prompt / 큰 docs) cache. token 으로는 차지하지만 비용 + latency 절감.

### Tool result 압축
```python
# Bad — large grep
result = grep(pattern, "**/*.py")    # 10000 lines

# Good — head + count
matches = grep(pattern, "**/*.py")
return f"{len(matches)} matches. First 20:\n" + "\n".join(matches[:20])
```

### File reading — pagination
```python
# 큰 file 의 일부만
read_file(path, offset=1000, limit=200)
```

### `/compact` (Claude Code)
사용자가 명시적으로 trigger — 옛 conversation 자동 요약.

## 진단

```python
# Token 예상 (Anthropic SDK)
from anthropic import Anthropic
client = Anthropic()
count = client.count_tokens(messages=messages, model="claude-sonnet-4-6")
print(count.input_tokens)
```

```
# Claude Code
/status — current context size
/cost — token 누적
```

## 재발 방지

- system prompt 는 짧게 (cache_control 활용)
- 큰 file / docs 는 retrieval (RAG) 으로 필요시만
- tool result 의 head/wc/pagination 패턴 강제
- sub-agent 우선 — 큰 search / analysis 위임
- 모니터링: token usage 추세

## 관련

- [[ai-agent]] / [[tool-result-too-large]] / [[multi-turn-forgetting]]
