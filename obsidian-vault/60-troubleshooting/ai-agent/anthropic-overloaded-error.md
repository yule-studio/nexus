---
title: "troubleshooting · Anthropic API overloaded_error 529"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, anthropic, api, retry, fallback]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]", "[[rate-limit-429]]"]
---

# Anthropic API `overloaded_error` 529

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
anthropic.OverloadedError: Error code: 529
{
  "type": "error",
  "error": {
    "type": "overloaded_error",
    "message": "Overloaded"
  }
}
```

- 사용량 정상인데 발생 (rate limit 아님)
- 특정 시간대 (피크) 에 spike
- 모델 capacity 일시 부족

## 원인

Anthropic 측 capacity 한계 (rate limit 과 다름):
- 모델 (특히 Opus) 의 GPU 공급 부족
- 글로벌 traffic spike
- 새 model launch 직후

→ 사용자 측 잘못 없음. Anthropic 의 일시적 상태.

## 해결

### Exponential backoff + retry
```python
import asyncio, random
from anthropic import AsyncAnthropic, OverloadedError

async def call_with_retry(messages, model="claude-opus-4-7", max_retries=5):
    for attempt in range(max_retries):
        try:
            return await client.messages.create(model=model, messages=messages)
        except OverloadedError:
            if attempt == max_retries - 1: raise
            wait = (2 ** attempt) + random.uniform(0, 2)
            await asyncio.sleep(wait)
```

→ 보통 30-60초 후 회복.

### Model fallback
```python
PRIMARY = "claude-opus-4-7"
FALLBACK = "claude-sonnet-4-6"

try:
    return await call(PRIMARY, ...)
except OverloadedError:
    logger.warn("primary overloaded, falling back")
    return await call(FALLBACK, ...)
```

→ Opus → Sonnet → Haiku 순. quality 일부 손실 but availability.

### Region / endpoint 다양화
Anthropic 은 single endpoint. AWS Bedrock / GCP Vertex AI 의 Claude 도 사용 가능:

```python
# AWS Bedrock
import boto3
bedrock = boto3.client("bedrock-runtime")
response = bedrock.invoke_model(
    modelId="anthropic.claude-sonnet-4-6-v1:0",
    body=json.dumps({...}),
)
```

→ Anthropic API 와 다른 capacity pool. 같은 시점에 한쪽만 overload 가능.

### Queue + worker (대량 batch)
```python
# 사용자 응답 latency 가 중요한 경우 fallback
# Batch job 은 queue + 천천히
async def batch_process():
    for task in tasks:
        while True:
            try:
                result = await llm.call(task)
                break
            except OverloadedError:
                await asyncio.sleep(60)   # 1분 대기
        save(result)
```

### Status page 모니터링
```
https://status.anthropic.com/
```

- API status webhook 받기
- 알람: degraded / outage 시 즉시 fallback mode

```python
# 자동 health check
import httpx

async def is_anthropic_healthy():
    r = await httpx.get("https://status.anthropic.com/api/v2/status.json")
    return r.json()["status"]["indicator"] == "none"
```

## 진단

```python
# Error rate
anthropic_errors_total{type="overloaded"}

# Response time spike
anthropic_request_duration_seconds{model="opus"}

# Anthropic status
curl https://status.anthropic.com/api/v2/status.json
```

## 재발 방지

- Retry + backoff 표준 (모든 API call 에 wrapper)
- Model fallback chain (Opus → Sonnet → Haiku)
- Bedrock / Vertex 백업 endpoint
- 사용자 facing latency 중요 시 timeout + degrade 응답
- 알람: overloaded rate > 5%

## 관련

- [[ai-agent]] / [[rate-limit-429]] / [[streaming-truncation]]
