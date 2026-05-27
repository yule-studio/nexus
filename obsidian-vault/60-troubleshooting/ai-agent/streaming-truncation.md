---
title: "troubleshooting · LLM streaming truncation"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, streaming, sse, max-tokens]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# Streaming 응답이 mid-token 에서 끊김

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
응답 텍스트: "Here is the answer to your question. The capital of"
(끝)
```

- 응답이 완전한 문장 / JSON / code block 못 끝냄
- `stop_reason: "max_tokens"` 또는 network disconnect

## 원인

1. `max_tokens` 너무 작음
2. 중간 network disconnect / proxy timeout
3. SSE keep-alive 끊김
4. LLM 자체 length limit

## 해결

### `max_tokens` 충분히
```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,    # default 1024 → 충분히 (Sonnet 4096+ OK)
    messages=messages,
)
```

→ Claude Opus 4.7 / Sonnet 4.6 는 8192 까지. response 가 정말 길면 streaming + continuation.

### `stop_reason` 검증
```python
async for event in stream:
    if event.type == "message_stop":
        if event.stop_reason == "max_tokens":
            # 잘림 → continuation
            messages.append({"role": "assistant", "content": full_text})
            messages.append({"role": "user", "content": "Continue from where you stopped."})
            # 재호출
```

### SSE / proxy keepalive
```nginx
location /stream {
    proxy_pass http://backend;
    proxy_buffering off;          # ← streaming 핵심
    proxy_cache off;
    proxy_read_timeout 300s;       # 응답 idle timeout 길게
    proxy_send_timeout 300s;
    chunked_transfer_encoding on;
    add_header X-Accel-Buffering no;
}
```

### Network disconnect 처리
```python
async def stream_with_retry(prompt, max_retries=3):
    accumulated = ""
    for attempt in range(max_retries):
        try:
            async with client.messages.stream(messages=[{"role": "user", "content": prompt}], max_tokens=8192) as s:
                async for chunk in s.text_stream:
                    accumulated += chunk
                    yield chunk
            return
        except (anthropic.APIConnectionError, asyncio.TimeoutError):
            if attempt == max_retries - 1: raise
            # continuation
            messages.append({"role": "assistant", "content": accumulated})
            messages.append({"role": "user", "content": "Continue."})
```

### Frontend SSE 처리
```ts
const evtSource = new EventSource('/api/stream');
evtSource.addEventListener('message', (e) => {
  appendText(e.data);
});
evtSource.addEventListener('error', () => {
  // 재연결 (브라우저 자동 또는 수동)
});
```

→ EventSource 가 자동 reconnect. 단 정확한 state 복구는 server 가 sequence id 제공.

### JSON 응답 — partial JSON parser
```python
# Streaming JSON 은 incomplete state 가능
import json_partial

parser = json_partial.Parser()
for chunk in stream:
    parser.feed(chunk)
    if parser.complete():
        return parser.value()
```

또는 명확한 delimiter:
```
{
  "items": [
    {"id": 1, "value": "..."}
    {"id": 2, "value": "..."}    ← 잘림
```

→ tool_use 형식 우선 (Anthropic 이 양쪽 완성 보장).

## 진단

```python
# 응답 metadata
print(response.stop_reason)   # end_turn / max_tokens / stop_sequence / tool_use
print(response.usage.output_tokens)
```

```
# Network
- Browser DevTools → Network → SSE event 시각
- gap 이 30초+ 면 proxy timeout 의심
```

## 재발 방지

- `max_tokens` 8192+ 표준 (긴 응답 가능)
- `stop_reason == "max_tokens"` 자동 continuation
- proxy / LB 의 `proxy_buffering off` + idle_timeout 길게
- 모니터링: stop_reason 분포 (max_tokens 비율 ↑ 시 max_tokens 늘림)

## 관련

- [[ai-agent]] / [[context-overflow]] / [[rate-limit-429]]
