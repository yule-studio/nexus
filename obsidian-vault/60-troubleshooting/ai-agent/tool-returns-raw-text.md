---
title: "troubleshooting · Tool 가 expected schema 아닌 raw text 반환"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [ai-agent, tool-use, schema, parsing]
home_hub: "60-troubleshooting"
related: ["[[ai-agent]]", "[[tool-malformed-json]]"]
---

# Tool response 가 free-form text — agent 파싱 실패

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | ai-agent | 최초 |

## 증상

```
Tool response:
"Found 5 results. Here they are:\n1. Doc A\n2. Doc B..."

Agent 가 파싱 시도:
- JSON.parse fail
- Regex 로 추출 시도 → fragile
- 다음 turn 의 분기가 잘못 추론
```

## 원인

Tool 가 정해진 schema 안 따르고 자연어 text 반환. LLM agent 가 raw text 에서 구조 추출하려다 실패 / hallucinate.

```python
# Bad — tool 가 text 반환
def search(query):
    matches = full_search(query)
    return f"Found {len(matches)}. First: {matches[0].title}"

# Agent 다음 turn — 두 번째 결과는?
```

## 해결

### Tool 가 항상 structured JSON
```python
def search(query):
    matches = full_search(query)
    return {
        "count": len(matches),
        "results": [
            {"id": m.id, "title": m.title, "snippet": m.snippet[:200], "score": m.score}
            for m in matches[:10]
        ],
        "has_more": len(matches) > 10,
    }
```

→ agent 가 `result["results"][0]["id"]` 같이 정확한 access.

### Response schema 명시
```python
{
    "name": "search",
    "input_schema": {...},
    # tool description 에 response 형식 명시
    "description": """검색.
응답 형식:
{
  "count": <int>,
  "results": [{"id": str, "title": str, "snippet": str, "score": float}],
  "has_more": bool
}""",
}
```

→ LLM 가 response 형식 알면 정확한 파싱.

### Parser at orchestrator
```python
@validate_response(SearchResponseSchema)
async def search(args):
    raw = call_search_api(args)
    return raw   # schema 검증 통과해야 agent 에 전달
```

```python
from pydantic import BaseModel

class SearchResponse(BaseModel):
    count: int
    results: list[SearchResult]
    has_more: bool

# Orchestrator
response = SearchResponse(**raw_response)   # validation
agent.receive_tool_result(response.model_dump())
```

### Legacy tool 의 wrapping
```python
# 옛 tool 가 text 반환 — wrapper 가 structured 로
def search_v2(query):
    text = legacy_search(query)
    # parse text → struct
    lines = text.split("\n")
    count_match = re.search(r"Found (\d+)", lines[0])
    return {
        "count": int(count_match.group(1)) if count_match else 0,
        "raw_text": text,
        "lines": lines,
    }
```

### 일관된 error 응답
```python
def safe_tool_call(args):
    try:
        return {"ok": True, "data": call_real_tool(args)}
    except Exception as e:
        return {"ok": False, "error": str(e), "error_type": type(e).__name__}
```

→ agent 가 `if response.ok: ... else: handle_error(...)` 정확히.

## MCP (Model Context Protocol) 활용

Anthropic 의 MCP — tool 의 response schema 가 spec 으로:

```json
{
  "method": "tools/call",
  "result": {
    "content": [
      {"type": "text", "text": "..."},
      {"type": "image", "data": "...", "mimeType": "image/png"}
    ],
    "isError": false
  }
}
```

→ tool 종류 (text / image / resource) 명확. agent 가 type 보고 처리.

## 진단

```python
# Tool response 의 type 분포
counter = Counter(type(r).__name__ for r in tool_responses)
# 비정상 — str 가 dict 보다 많으면 schema 안 따름

# Response validation error
- pydantic_validation_error_total{tool="search"}
```

## 재발 방지

- 모든 tool 가 structured response (dict / JSON) 표준
- Tool description 에 response schema 명시
- Pydantic / JSON Schema validator at orchestrator
- 새 tool 추가 시 schema test fixture
- MCP 표준 도입 (가능 시)

## 관련

- [[ai-agent]] / [[tool-malformed-json]] / [[function-call-schema-drift]]
