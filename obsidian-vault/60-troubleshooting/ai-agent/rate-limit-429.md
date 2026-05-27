---
title: "troubleshooting · LLM API Rate limit 429"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, anthropic, openai, api, rate-limit]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# LLM API — `429 Too Many Requests`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
anthropic.RateLimitError: Error code: 429
{
  "error": {
    "type": "rate_limit_error",
    "message": "Number of request tokens has exceeded your rate limit (...)"
  }
}
```

또는:
```
HTTP 429 Too Many Requests
Retry-After: 60
```

## 원인

API provider 의 rate limit:
- 분당 token (TPM)
- 분당 request (RPM)
- 일 한도 (TPD)

Burst spike 또는 parallel call 폭증 시 도달.

## 해결

### Exponential backoff + jitter
```python
import asyncio, random

async def call_with_retry(func, *args, max_retries=5, **kwargs):
    for attempt in range(max_retries):
        try:
            return await func(*args, **kwargs)
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            # Retry-After header 우선
            wait = int(e.response.headers.get("retry-after", 0))
            if wait == 0:
                wait = (2 ** attempt) + random.uniform(0, 1)
            await asyncio.sleep(wait)
```

### Parallel limit (semaphore)
```python
import asyncio

semaphore = asyncio.Semaphore(10)   # 최대 10 동시 call

async def safe_call(args):
    async with semaphore:
        return await llm.call(args)
```

### Token bucket — application level
```python
from aiolimiter import AsyncLimiter

limiter = AsyncLimiter(max_rate=1000, time_period=60)  # 분당 1000 token

async def call(prompt):
    tokens = estimate_tokens(prompt)
    await limiter.acquire(tokens)
    return await llm.call(prompt)
```

### Caching (반복 query)
```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_summarize(text):
    return llm.summarize(text)
```

또는 Anthropic `cache_control` — static portion 만 cache:
```python
messages = [{"role": "user", "content": [
    {"type": "text", "text": large_docs, "cache_control": {"type": "ephemeral"}},
    {"type": "text", "text": user_q},
]}]
```

→ token 으로 계산되지만 cost 90% 절감, 빠름.

### Model fallback
```python
try:
    return await opus.call(...)
except RateLimitError:
    return await sonnet.call(...)   # fallback
```

### Queue + worker
```python
# 대량 task — sync 호출 X
queue.put(task)

# worker
while True:
    task = queue.get()
    try:
        result = await llm.call(task)
    except RateLimitError:
        queue.put(task)   # requeue
        await asyncio.sleep(60)
```

### Provider 의 tier 상향
- Anthropic / OpenAI 의 tier (사용량 기반) 자동 증가
- 또는 enterprise plan 협상

## 진단

```python
# 응답 headers
response.headers.get("anthropic-ratelimit-requests-remaining")
response.headers.get("anthropic-ratelimit-tokens-remaining")
response.headers.get("anthropic-ratelimit-tokens-reset")
response.headers.get("retry-after")
```

```
# 사용량 dashboard
# Anthropic: https://console.anthropic.com/settings/limits
# OpenAI: https://platform.openai.com/usage
```

## 재발 방지

- Exponential backoff + jitter 표준 wrapper
- Parallel call 에 semaphore 의무
- Prompt caching 활용 (static 부분)
- 모니터링: 429 rate / token usage / TPM 추세
- 알람: usage 가 tier 의 80% 도달

## 관련

- [[ai-agent]] / [[anthropic-overloaded-error]] / [[context-overflow]]
