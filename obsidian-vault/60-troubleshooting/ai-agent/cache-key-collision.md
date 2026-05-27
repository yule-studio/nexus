---
title: "troubleshooting · Cache key collision"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, cache, anthropic, hash]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]"]
---

# Cache key collision — 다른 prompt 가 같은 응답

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

- 다른 사용자 / 다른 query 인데 같은 답변
- prompt cache 적중률 비정상 높음 (정상 ~50%)
- 응답이 user_id 와 무관한 내용

## 원인

prompt cache key 가 hash 인데 hash input 이 prompt 전체 context 포괄 X. 예:
- key = hash(prompt) 만 — system / user / tools 미포함
- key = hash(user_message) 만 — system 변경 영향 X

## 해결

### Cache key 에 모든 input 포함
```python
import hashlib, json

def cache_key(system, messages, tools, model, temperature):
    payload = {
        "system": system,
        "messages": messages,
        "tools": [t["name"] + ":" + tool_hash(t) for t in tools],
        "model": model,
        "temperature": temperature,
    }
    return hashlib.sha256(json.dumps(payload, sort_keys=True).encode()).hexdigest()
```

### User isolation
```python
# Bad — global cache (user 간 공유)
cache.get(key)

# Good — user namespace
cache.get(f"u:{user_id}:{key}")
```

→ 다른 user 가 같은 prompt 라도 별도 cache. PII / 개인 정보 leak 방지.

### Anthropic `cache_control` 정확히
```python
messages = [{
    "role": "user",
    "content": [
        {"type": "text", "text": SYSTEM_DOC, "cache_control": {"type": "ephemeral"}},   # static
        {"type": "text", "text": user_question},   # variable — cache 안 함
    ],
}]
```

→ static 부분만 cache. variable 부분이 매번 달라야 정확한 응답.

### TTL — stale 결과 차단
```python
# In-memory cache
from cachetools import TTLCache
cache = TTLCache(maxsize=10000, ttl=300)   # 5분
```

→ 옛 응답이 영원히 살지 않음.

### Hash 충돌 (희박하지만 가능)
SHA-256 의 충돌 가능성 사실상 0 — md5 / shorter hash 사용 시만 위험. SHA-256 권장.

## 진단

```python
# Cache 적중률
cache_hits / total_requests

# 같은 key 의 응답 변화
sample_keys = random.sample(cache.keys(), 100)
for k in sample_keys:
    print(k, cache[k][:50])

# 비정상 패턴 — 다른 user 가 같은 cache key
SELECT cache_key, COUNT(DISTINCT user_id) FROM cache_log
GROUP BY cache_key HAVING COUNT(DISTINCT user_id) > 1;
```

## Anthropic prompt cache 검증

```python
response = client.messages.create(...)
print(response.usage.cache_read_input_tokens)    # cache hit
print(response.usage.cache_creation_input_tokens)  # cache miss (생성)
```

→ cache_read 가 0 이면 cache 안 작동. cache_creation 이 일정하게 발생 → static 부분이 매번 새로 등록.

## 재발 방지

- Cache key 에 모든 input (system / user / tools / model / temp / user_id)
- TTL 표준 (5-60분)
- 모니터링: 같은 cache key 의 user 분포
- 보안: PII 가 cache 안 들어가는지 audit

## 관련

- [[ai-agent]] / [[function-call-schema-drift]] / [[rate-limit-429]]
