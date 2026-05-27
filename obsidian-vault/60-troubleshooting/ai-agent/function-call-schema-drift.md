---
title: "troubleshooting · Function call schema drift"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, tool-use, schema, versioning]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]", "[[tool-malformed-json]]"]
---

# Schema drift — 새 버전이 옛 호출 거부

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

- Agent (or LLM cache) 가 옛 schema 의 tool 호출
- Server 가 새 schema 거절 → unknown field / missing required
- 같은 코드가 어제는 정상, 오늘 failure (deploy 직후)

## 원인

Tool / function schema 가 backward incompatible 변경:
- 새 required field 추가
- field rename
- enum value 제거
- type 변경 (string → int)

```python
# 옛 schema
{"name": "search", "input": {"query": "x"}}

# 새 schema (required field 추가)
{"name": "search", "input": {"query": "x", "tenant_id": "..."}}   # required
```

옛 LLM 응답 cache 가 옛 schema 그대로 호출 → 거절.

## 해결

### Version 명시 — name 에 포함
```python
tools = [
    {"name": "search_v1", "input_schema": V1_SCHEMA},   # legacy 한동안 유지
    {"name": "search_v2", "input_schema": V2_SCHEMA},   # new
]
```

→ 새 LLM 호출은 v2, 옛 v1 도 작동. v1 deprecation 일정 명시.

### Backward-compatible 필드 추가
```python
# Bad — required 추가 → 옛 호출 break
input_schema["required"].append("tenant_id")

# Good — optional + default
input_schema["properties"]["tenant_id"] = {"type": "string", "default": "default-tenant"}
# 또는 server 측에서 derive
```

### Rename → alias 유지
```python
# 옛 field "query" → 새 "q"
def search(args):
    query = args.get("q") or args.get("query")   # 양쪽 받음
    if not query: raise ValueError("missing query")
```

### Enum 추가 OK, 제거 X
```python
# 옛: ["doc", "code"]
# 새: ["doc", "code", "issue"]   ← OK (옛 호출도 valid)
# 새: ["doc"]                    ← break (옛 "code" 호출 거절)
```

### Schema 변경 시 LLM cache invalidate
```python
import hashlib

def tool_hash(tool):
    return hashlib.md5(json.dumps(tool["input_schema"], sort_keys=True).encode()).hexdigest()

# Prompt cache key 에 tool hash 포함 → schema 변경 시 cache miss
```

## Spec 관리

```yaml
# OpenAPI / JSON Schema
# tools/search.v1.json
# tools/search.v2.json

# CI 가 backward compat 검증
# - new required field → fail
# - removed enum value → fail
```

OpenAPI Diff / openapi-cli-diff 같은 도구 활용.

## Server-side 처리

```python
@router.post("/tool/search")
def search(req: SearchRequest):
    # Pydantic 가 validation
    # → unknown field 무시 (extra = "ignore") 또는 reject (extra = "forbid")
    pass

class SearchRequest(BaseModel):
    query: str
    tenant_id: str | None = None   # optional with default

    model_config = ConfigDict(extra="ignore")   # 옛 호출의 extra field 무시
```

## Yule Studio Agent 사례

slash command schema 변경 시:
- 옛 turn 의 tool call 이 invalid 가 됨
- session recovery 시 옛 schema 의 call 을 retry → 거절
- → schema change 시 session migration helper

## 진단

```python
# Tool schema 변경 사항
diff tools/search.v1.json tools/search.v2.json

# Validation error sample
logger.warn("tool_call_rejected", tool=name, args=args, error=e)
```

## 재발 방지

- Tool name 에 version (`search_v1`) — major change 시 새 name
- Field 추가는 optional + default
- Field 제거 / rename 시 alias / migration 기간 (3개월)
- Spec 관리: openapi-cli-diff in CI

## 관련

- [[ai-agent]] / [[tool-malformed-json]] / [[cache-key-collision]]
